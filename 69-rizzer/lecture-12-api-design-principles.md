# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONSUMER FIRST, IMPLEMENTATION SECOND                          │
│  ─────────────────────────────────────                          │
│  Students must think like the CALLER of their API, not the      │
│  builder. Every design decision starts with: "If I were the     │
│  client consuming this, what would I expect?"                   │
│                                                                 │
│  PAIN BEFORE PRINCIPLE                                          │
│  ────────────────────                                           │
│  Show a BADLY designed API first. Let them struggle with it.    │
│  Then introduce each principle as the cure for a specific pain. │
│                                                                 │
│  TRADEOFFS, NOT RULES                                           │
│  ────────────────────                                           │
│  API design is full of "it depends." Every pattern has a cost.  │
│  We teach WHEN to use each pattern, not just HOW.               │
│                                                                 │
│  CONNECT TO THEIR PROJECT                                       │
│  ────────────────────────                                       │
│  All examples use the Task Manager API they are already         │
│  building. By the end, they know exactly what to add.           │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ────────────────────────                                       │
│  Pydantic → Pagination/filter response models                   │
│  Depends() → Reusable pagination/filter parameters              │
│  HTTP/REST → Status codes, method semantics, resources          │
│  Testing → They'll test every pattern they learn here           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                   API DESIGN PRINCIPLES                         │
│                     (5-6 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (30 min)                                   │
│  ├─ 1.1 The API From Hell (Demonstration)        [UPDATED]      │
│  ├─ 1.2 What Makes an API "Good"?                [UPDATED]      │
│  └─ 1.3 The Online Store Analogy                 [UPDATED]      │
│                                                                 │
│  PART 2: API VERSIONING (55 min)                                │
│  ├─ 2.1 Why Version? (The Breaking Change)                      │
│  ├─ 2.2 URL Path Versioning                                     │
│  ├─ 2.3 Header Versioning                                       │
│  ├─ 2.4 Query Parameter Versioning                              │
│  ├─ 2.5 The Verdict: Which Strategy Wins?                       │
│  └─ 2.6 Implementing in FastAPI             [FULL REPLACEMENT]  │
│       ├─ Tech Debt: Dev setup (uvicorn + debug config)          │
│       ├─ Tech Debt: include_router() full parameters            │
│       └─ Input/Output Model Separation (Mass Assignment)        │
│                                                                 │
│  PART 3: PAGINATION (50 min)                                    │
│  ├─ 3.1 The "Dump Everything" Problem                           │
│  ├─ 3.2 Offset-Based Pagination (The Simple Way)                │
│  ├─ 3.3 The Shifting Window Trap                                │
│  ├─ 3.4 Cursor-Based Pagination (The Robust Way)                │
│  ├─ 3.5 Offset vs Cursor: The Decision                          │
│  └─ 3.6 Response Envelope Design (Connection to Pydantic)       │
│                                                                 │
│  PART 4: FILTERING AND SORTING (55 min)                         │
│  ├─ 4.1 The Client Shouldn't Do Your Job                        │
│  ├─ 4.2 Filtering Conventions                                   │
│  ├─ 4.3 Sorting Conventions                                     │
│  ├─ 4.4 Combining Everything (Connection to Depends)            │
│  ├─ 4.5 Nested Resources vs Query Parameters         [NEW]      │
│  └─ 4.6 Ownership-Aware Filtering                    [NEW]      │
│                                                                 │
│  PART 5: IDEMPOTENCY (45 min)                                   │
│  ├─ 5.1 What Is Idempotency?                                    │
│  ├─ 5.2 HTTP Methods and Idempotency                            │
│  ├─ 5.3 The Double-Click Disaster                               │
│  ├─ 5.4 Idempotency Keys (The Safety Net)                       │
│  └─ 5.5 Partial Updates with PATCH                   [NEW]      │
│                                                                 │
│  PART 6: HATEOAS & API AS CONTRACT (20 min)                     │
│  ├─ 6.1 HATEOAS: Teaching Your API to Give Directions           │
│  ├─ 6.2 API Documentation Is a Contract                         │
│  └─ 6.3 OpenAPI: The Contract You Already Have                  │
│                                                                 │
│  PART 7: STANDARDIZED ERROR ENVELOPES (35 min)       [NEW]      │
│  ├─ 7.1 The Missing Half of Your API                            │
│  ├─ 7.2 RFC 7807: Problem Details Standard                      │
│  └─ 7.3 Implementing Error Envelopes in FastAPI                 │
│                                                                 │
│  PART 8: HTTP CACHING (35 min)                       [NEW]      │
│  ├─ 8.1 The "Why Fetch Again?" Problem                          │
│  ├─ 8.2 ETag: The Fingerprint                                   │
│  └─ 8.3 Implementing ETags in FastAPI                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The API From Hell

**Start with a demonstration. Make them consume a terrible API.**

Give students this scenario:

> "You've been hired to build a frontend dashboard. Here's the API you have to work with. Fetch the tasks and display them."

```python
# terrible_api.py — The API nobody wants to use
from fastapi import FastAPI

app = FastAPI()

# Pretend this is a database with 50,000 tasks
TASKS: list[dict] = [
    {
        "id": i,
        "t": f"Task {i}",                    # What does "t" mean?
        "s": "done" if i % 3 == 0 else "pending",  # "s"? Status? Size? Score?
        "p": i % 5,                           # "p" for...?
        "cat": f"category-{i % 10}",
        "ts": 1700000000 + i,                 # Timestamp... in what format?
    }
    for i in range(50_000)
]


@app.get("/getTasks")                         # GET already implies "get"...
async def get_tasks():
    return TASKS                              # ALL 50,000. Every. Single. Time.


@app.get("/getTask")                          # Where's the task ID?
async def get_task(task_id: int):
    return TASKS[task_id]


@app.post("/deleteTask")                      # POST for delete??
async def delete_task(task_id: int):
    del TASKS[task_id]                        # Shifts all indexes!
    return {"msg": "ok"}                      # "msg"? Status code?


@app.post("/updateTask")                      # How is this different from create?
async def update_task(task_id: int, t: str):
    TASKS[task_id]["t"] = t
    return TASKS[task_id]
```

**Now run it. Hit `GET /getTasks` in the browser.**

```
Response size: ~4.2 MB of JSON
Response time: 1800ms
Browser: freezing, scrollbar broken, dev tools crying
```

**Ask the class:**

> "You're a frontend developer. It's Monday morning. Your manager says 'just show the first 20 tasks.' What do you do with this API?"

Answer: **You download all 50,000 tasks and throw away 49,980 of them.** Every. Single. Request.

> "Now your manager says 'let users filter by status.' What do you do?"

Answer: **Download all 50,000 AGAIN, then filter in JavaScript.** On the user's phone. On 3G.

> "Now your manager says 'we redesigned the task format, update the frontend.' But the mobile app team hasn't updated yet."

Answer: **You change the API response format and the mobile app crashes.** Two teams pointing fingers.

```
┌─────────────────────────────────────────────────────────────────┐
│                     WHAT WENT WRONG?                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM 1: No pagination                                       │
│  └─ 50,000 items in one response. Server melts. Client melts.   │
│                                                                 │
│  PROBLEM 2: No filtering or sorting                             │
│  └─ Client does the server's job. Wastes bandwidth and CPU.     │
│                                                                 │
│  PROBLEM 3: No versioning                                       │
│  └─ One change breaks all consumers. No way to evolve safely.   │
│                                                                 │
│  PROBLEM 4: Wrong HTTP methods (POST for delete)                │
│  └─ Violates REST. Breaks caching. Confuses everyone.           │
│                                                                 │
│  PROBLEM 5: Cryptic field names (t, s, p)                       │
│  └─ No documentation. Developers guess and get it wrong.        │
│                                                                 │
│  PROBLEM 6: Deleting by index shifts everything                 │
│  └─ Delete task 5, now task 6 is task 5. Non-idempotent chaos.  │
│                                                                 │
│  PROBLEM 7: No error structure                                  │
│  └─ Success returns {"msg": "ok"}. Failure also returns         │
│     {"msg": "ok"} or {"msg": "error"}. No status codes.         │
│     No field-level detail. Client can't tell what went wrong    │
│     or how to fix it.                                           │
│                                                                 │
│  PROBLEM 8: No user ownership                                   │
│  └─ GET /getTasks returns ALL 50,000 tasks from ALL users.      │
│     No authentication context. User A's private tasks are       │
│     fully visible to User B. This isn't a design flaw —         │
│     it's a security breach hiding as an API.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The insight:**

> "Every one of these problems has a name and a solution. That's what this lecture is about. By the end, you'll look at that API and know exactly how to fix every single issue — and you'll apply those fixes to your own Task Manager."

---

## 1.2 What Makes an API "Good"?

```
┌─────────────────────────────────────────────────────────────────┐
│                  QUALITIES OF A GOOD API                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. PREDICTABLE                                                 │
│     If I know how GET /tasks works, I can guess                 │
│     how GET /categories works. Consistent conventions.          │
│                                                                 │
│  2. EVOLVABLE                                                   │
│     I can add features and change formats without               │
│     breaking existing consumers. Versioning.                    │
│                                                                 │
│  3. EFFICIENT                                                   │
│     I only get the data I need. Not more, not less.             │
│     Pagination, filtering, field selection.                     │
│                                                                 │
│  4. SAFE TO RETRY                                               │
│     If my network hiccups and I send a request twice,           │
│     nothing bad happens. Idempotency.                           │
│                                                                 │
│  5. SELF-DESCRIBING                                             │
│     The API tells me what it can do, what it expects,           │
│     and where to go next. Documentation + HATEOAS.              │
│                                                                 │
│  6. CONSISTENT IN FAILURE                                       │
│     When something goes wrong, the error looks the same         │
│     every time. Machine-readable. Structured. Actionable.       │
│     Not {"msg": "error"} — but a proper error shape with        │
│     type, title, status code, and a human-readable detail.      │
│                                                                 │
│  7. CACHE-FRIENDLY                                              │
│     I can tell whether data has changed without re-downloading  │
│     it. Saves bandwidth on every read. Mandatory for mobile     │
│     clients where bandwidth is expensive.                       │
│                                                                 │
│  8. OWNERSHIP-AWARE                                             │
│     The API knows WHO is asking. My data is mine alone.         │
│     An unauthenticated request or a wrong-user request          │
│     never returns private data. No exception.                   │
│                                                                 │
│  These eight qualities map directly to Parts 2–8.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**These eight qualities map directly to Parts 2–8.**

---

## 1.3 The Online Store Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE ONLINE STORE ANALOGY                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your API is an ONLINE STORE.                                   │
│  Your endpoints are the PAGES.                                  │
│  Your consumers (frontend, mobile, other services)              │
│  are the SHOPPERS browsing your store.                          │
│                                                                 │
│                                                                 │
│  BAD STORE (our terrible API):                                  │
│  ─────────────────────────────                                  │
│  • One giant page showing ALL 50,000 products at once           │
│  • No categories, no search, no filters                         │
│  • The store layout changes overnight with no warning           │
│  • Clicking "Buy" twice charges your card twice                 │
│  • No signs, no help desk, no store directory                   │
│  • Error page says "Something went wrong." That's it.           │
│    No detail, no error code, no suggestion. ← No error shape   │
│  • Any shopper can browse any other customer's order history     │
│    just by changing a number in the URL. ← No ownership         │
│  • Every product page reloads from scratch on every visit,      │
│    even if the price hasn't changed. ← No caching               │
│  • The checkout form lets you submit any price you want         │
│    and the server honours it. ← No input/output separation      │
│                                                                 │
│                                                                 │
│  GOOD STORE (what we're building):                              │
│  ─────────────────────────────────                              │
│  • Products shown 20 per page with "Next" button (PAGINATION)   │
│  • Sidebar filters: price, category, rating (FILTERING)         │
│  • Sort by: price low→high, newest, popular (SORTING)           │
│  • Old bookmarks still work after a redesign (VERSIONING)       │
│  • Clicking "Buy" twice only charges once (IDEMPOTENCY)         │
│  • "You might also like..." links (HATEOAS)                     │
│  • Clear help section and store directory (DOCUMENTATION)       │
│  • "Item not found" page tells you the item ID, explains why    │
│    it's gone, and links to similar items. (ERROR ENVELOPES)     │
│  • Your order history requires login. No one else can see       │
│    it by guessing a URL. (OWNERSHIP-AWARE ENDPOINTS)            │
│  • Browser uses its cached product page when the price          │
│    hasn't changed — no unnecessary re-download. (HTTP CACHING)  │
│  • Checkout only accepts quantity. Price is always server-set.  │
│    You cannot submit a price field and have it honoured.        │
│    (INPUT/OUTPUT SEPARATION)                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to API design:**

```
Online Store                │  API Design
────────────────────────────│──────────────────────────────
Store pages / catalog       │  API endpoints
Shoppers / customers        │  API consumers (frontend, etc.)
Products per page (20)      │  Pagination (limit, cursor)
Sidebar filters             │  Query parameter filtering
Sort dropdown               │  Sorting conventions
Store redesign / new layout │  API version change
"Buy" button safety         │  Idempotent operations
"Related items" links       │  HATEOAS (links in responses)
Store directory / help desk │  OpenAPI documentation
"Something went wrong" error page   │  Standardized error envelopes
  with code, reason, and next steps │    (RFC 7807 Problem Details)
────────────────────────────────────│──────────────────────────────
Browser not re-downloading          │  HTTP ETags (Cache headers)
  unchanged product images          │
────────────────────────────────────│──────────────────────────────
Your private order history          │  Ownership-aware endpoints
  (login required to view)          │    (user context on all reads)
────────────────────────────────────│──────────────────────────────
Store sets its own prices;          │  Input/Output model separation
  customer can only pick quantity   │    (TaskCreate vs TaskResponse)
```

---

# PART 2: API VERSIONING

## 2.1 Why Version? (The Breaking Change)

**Scenario: your Task Manager API is live. Two teams consume it.**

```python
# Version 1: Your current Task Manager response
{
    "id": 1,
    "title": "Write unit tests",
    "status": "pending",
    "priority": 3
}
```

**Now your manager says: "We need to split `priority` into `priority_level` and `priority_label`, and nest the status inside a `state` object."**

```python
# Version 2: The new format your manager wants
{
    "id": 1,
    "title": "Write unit tests",
    "state": {
        "status": "pending",
        "updated_at": "2025-01-15T10:30:00Z"
    },
    "priority_level": 3,
    "priority_label": "medium"
}
```

**What happens if you just change it?**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE BREAKING CHANGE DISASTER                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE THE CHANGE:                                             │
│                                                                 │
│    Web Frontend ──GET /tasks──▶ API ──▶ {"status": "pending"}   │
│    Mobile App ────GET /tasks──▶ API ──▶ {"status": "pending"}   │
│                                                                 │
│    Everyone happy. ✅                                            │
│                                                                 │
│                                                                 │
│  YOU DEPLOY THE CHANGE:                                         │
│                                                                 │
│    Web Frontend ──GET /tasks──▶ API ──▶ {"state": {"status":…}} │
│         │                                                       │
│         └─▶ task.status → undefined  💥 CRASH                   │
│                                                                 │
│    Mobile App ────GET /tasks──▶ API ──▶ {"state": {"status":…}} │
│         │                                                       │
│         └─▶ task.status → undefined  💥 CRASH                   │
│              (App already in App Store. Can't force update.)    │
│                                                                 │
│  Everyone broken. ❌                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The insight:**

> "Your API is a **contract**. When someone builds a client that reads `task.status`, they're relying on that field existing. Changing it is like a store moving all its aisles overnight without telling anyone. Versioning lets you redesign while keeping the old layout available."

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT IS A "BREAKING CHANGE"?                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BREAKING (requires new version):                               │
│  ├─ Removing a field from a response                            │
│  ├─ Renaming a field                                            │
│  ├─ Changing a field's type (string → object)                   │
│  ├─ Removing an endpoint                                        │
│  ├─ Making an optional request field required                   │
│  └─ Changing the meaning of a status code                       │
│                                                                 │
│  NON-BREAKING (safe without new version):                       │
│  ├─ Adding a new field to a response                            │
│  ├─ Adding a new endpoint                                       │
│  ├─ Adding an optional request parameter                        │
│  ├─ Adding a new enum value (usually — careful!)                │
│  └─ Fixing a bug in existing behavior                           │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  If any existing client code would break → it's breaking.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 URL Path Versioning

**The most common approach. The version lives in the URL.**

```
GET /api/v1/tasks         ← Old clients use this
GET /api/v2/tasks         ← New clients use this
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   URL PATH VERSIONING                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  https://api.example.com/v1/tasks                               │
│                           ──                                    │
│                           └─ Version in the URL path            │
│                                                                 │
│  HOW IT WORKS:                                                  │
│  ├─ /v1/tasks → returns old format (status as string)           │
│  ├─ /v2/tasks → returns new format (status as object)           │
│  └─ Both exist simultaneously                                   │
│                                                                 │
│  STORE ANALOGY:                                                 │
│  Like having two floors in a store.                             │
│  Floor 1 has the old layout. Floor 2 has the new layout.        │
│  Shoppers pick which floor to visit.                            │
│                                                                 │
│                                                                 │
│  ✅ PROS:                          ❌ CONS:                     │
│  ├─ Dead simple to understand      ├─ URL gets noisy            │
│  ├─ Visible in browser             ├─ Bookmarks break on        │
│  ├─ Easy to route and test         │  version change             │
│  ├─ Easy to deprecate              ├─ Temptation to version     │
│  └─ Most APIs use this             │  too often                  │
│                                    └─ Duplicated route logic     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Used by:** GitHub API (`/v3/`), Stripe (`/v1/`), Twilio (`/2010-04-01/`), Google Maps (`/v1/`).

---

## 2.3 Header Versioning

**The version lives in the HTTP header. The URL stays clean.**

```
GET /api/tasks
Accept: application/vnd.myapi.v2+json
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    HEADER VERSIONING                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The URL is always:  GET /api/tasks                             │
│                                                                 │
│  The version is in the header:                                  │
│  ├─ Accept: application/vnd.myapi.v1+json  → old format        │
│  ├─ Accept: application/vnd.myapi.v2+json  → new format        │
│  └─ (no header)                            → default version   │
│                                                                 │
│  STORE ANALOGY:                                                 │
│  Like showing your membership card at the door.                 │
│  Gold card members see the premium layout.                      │
│  Regular shoppers see the standard layout.                      │
│  Same entrance either way.                                      │
│                                                                 │
│                                                                 │
│  ✅ PROS:                          ❌ CONS:                     │
│  ├─ Clean URLs                     ├─ Invisible in browser      │
│  ├─ URLs don't change              ├─ Harder to test manually   │
│  ├─ Proper HTTP semantics          ├─ Easy to forget the header │
│  └─ One URL = one resource         ├─ More complex routing      │
│                                    └─ Harder to share links     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Used by:** GitHub API (also supports `Accept` header), Azure DevOps.

---

## 2.4 Query Parameter Versioning

**The version lives in the query string.**

```
GET /api/tasks?version=2
```

```
┌─────────────────────────────────────────────────────────────────┐
│                QUERY PARAMETER VERSIONING                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GET /api/tasks?version=1          → old format                 │
│  GET /api/tasks?version=2          → new format                 │
│  GET /api/tasks                    → default (latest? oldest?)  │
│                                                                 │
│  STORE ANALOGY:                                                 │
│  Like adding "?layout=classic" to the store URL.                │
│  Same store, different presentation based on your preference.   │
│                                                                 │
│                                                                 │
│  ✅ PROS:                          ❌ CONS:                     │
│  ├─ Easy to add                    ├─ Mixes versioning with     │
│  ├─ Visible in URL                 │  regular query params      │
│  ├─ Easy to default                ├─ What if someone forgets   │
│  └─ Low implementation effort      │  the param?                │
│                                    ├─ Pollutes caching          │
│                                    └─ Uncommon in practice      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Used by:** Google APIs (some), AWS (some).

---

## 2.5 The Verdict: Which Strategy Wins?

```
┌─────────────────────────────────────────────────────────────────┐
│                  COMPARISON TABLE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│              │ URL Path   │ Header    │ Query Param              │
│  ────────────┼────────────┼───────────┼──────────────           │
│  Simplicity  │ ★★★★★      │ ★★★       │ ★★★★                    │
│  Visibility  │ ★★★★★      │ ★★        │ ★★★★                    │
│  Clean URLs  │ ★★         │ ★★★★★     │ ★★★                     │
│  Cacheability│ ★★★★★      │ ★★★       │ ★★★                     │
│  Adoption    │ ★★★★★      │ ★★★       │ ★★                      │
│  ────────────┼────────────┼───────────┼──────────────           │
│  VERDICT     │ USE THIS   │ Know it   │ Avoid usually           │
│              │ FIRST      │ exists    │                          │
│                                                                 │
│                                                                 │
│  FOR THIS COURSE AND YOUR PROJECT:                              │
│  ─────────────────────────────────                              │
│  Use URL path versioning. It's the industry standard            │
│  for most REST APIs. It's explicit, testable, and every         │
│  developer understands it immediately.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "When should you create v2? Should you version from day one?"

Answer:

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHEN TO VERSION                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ START with /v1/ from the beginning.                         │
│     It costs nothing and saves you pain later.                  │
│                                                                 │
│  ✅ Create /v2/ ONLY when you have a BREAKING change.           │
│     Non-breaking changes go into the current version.           │
│                                                                 │
│  ❌ DON'T create a new version for every small change.          │
│     v47 means you have 47 versions to maintain.                 │
│                                                                 │
│  ❌ DON'T keep old versions alive forever.                      │
│     Announce deprecation → give migration time → remove.        │
│                                                                 │
│  TYPICAL LIFECYCLE:                                             │
│     v1 (active) → v2 (active), v1 (deprecated) →               │
│     v2 (active), v1 (sunset) → v2 (active), v1 (removed)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.6 Implementing in FastAPI

---

### ⚙️ Tech Debt Payoff: Dev Setup Before You Code

> *Paying off two overdue items from Week 2, Lecture 1.*

Every time you build or debug FastAPI code in this course, you will run it one of two ways. Know both cold.

**Running for development (auto-reload on file save):**

```bash
# Minimal — just run it
uvicorn src.main:app

# Development mode — restarts automatically when you save a file
uvicorn src.main:app --reload

# Full development flags: expose on all interfaces, explicit port, verbose logs
uvicorn src.main:app --reload --host 0.0.0.0 --port 8000 --log-level debug
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    UVICORN FLAGS TO KNOW                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  --reload           Restart on code change (dev only!)          │
│                     Never use --reload in production.           │
│                     It watches the filesystem continuously.     │
│                                                                 │
│  --host 0.0.0.0     Accept connections from any interface       │
│                     Default is 127.0.0.1 (localhost only).      │
│                     Required when running inside Docker.         │
│                                                                 │
│  --port 8000        Listen on port 8000 (the default)           │
│                                                                 │
│  --log-level debug  Print every request, including headers      │
│                     Useful when debugging request/response       │
│                                                                 │
│  --workers 4        Run 4 worker processes (production only)    │
│                     Cannot combine with --reload                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Debugging with VS Code (`.vscode/launch.json`):**

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "FastAPI: Debug",
            "type": "debugpy",
            "request": "launch",
            "module": "uvicorn",
            "args": [
                "src.main:app",
                "--reload",
                "--port",
                "8000"
            ],
            "jinja2": true,
            "justMyCode": false
        },
        {
            "name": "Pytest: Debug Tests",
            "type": "debugpy",
            "request": "launch",
            "module": "pytest",
            "args": ["-v", "--tb=short"],
            "justMyCode": false
        }
    ]
}
```

> `justMyCode: false` is critical. Without it, the debugger refuses to step into FastAPI and Pydantic internals, which is exactly where confusing bugs often live.

> With `"name": "FastAPI: Debug"` selected in the Run panel, press **F5** to start the server with a live debugger attached. Set breakpoints in your route handlers by clicking the gutter. The server reloads on save, and the debugger reattaches automatically.

---

**Connection to what you've learned — the existing version code is unchanged below:**

```python
# src/api/v1/tasks.py  ← V1 code remains identical to original lecture
# src/api/v2/tasks.py  ← V2 code remains identical to original lecture
# src/main.py          ← Wire-up code shown below is being EXTENDED
```

*[The V1 router, V2 router, and original `main.py` content from the previous version of this section remain intact here.]*

---

### ⚙️ Tech Debt Payoff: `include_router()` Full Parameters

> *Paying off tech debt from Week 3, Lecture 4.*

Previously you set `prefix` and `tags` inside the router itself and called `app.include_router(router)` with no arguments. That works, but you were using only a fraction of `include_router()`'s power.

```
┌─────────────────────────────────────────────────────────────────┐
│            include_router() FULL SIGNATURE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  app.include_router(                                            │
│      router,           ← The APIRouter instance                 │
│      prefix=...,       ← PREPENDED to the router's own prefix   │
│      tags=[...],       ← REPLACES the router's own tags         │
│      responses={...},  ← Shared response schemas (OpenAPI docs) │
│      dependencies=[...],← Router-level deps (Week 9 preview)    │
│  )                                                              │
│                                                                 │
│  PREFIX STACKING:                                               │
│  ├─ Router prefix: ""          + include prefix: "/v1/tasks"    │
│  │   → Final path: /v1/tasks/...                                │
│  │                                                              │
│  ├─ Router prefix: "/tasks"   + include prefix: "/v1"           │
│  │   → Final path: /v1/tasks/...                                │
│  │                                                              │
│  └─ Router prefix: "/v1/tasks" + include prefix: ""             │
│      → Final path: /v1/tasks/... (current approach)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Pattern: Nested routers for clean version management**

This is the structure you will use as your API grows. Instead of baking `/v1` into every router, you create one version router that organises all resources underneath it.

```python
# src/api/v1/tasks.py
from fastapi import APIRouter

# Router knows nothing about versions — it just owns /tasks
tasks_router = APIRouter()

@tasks_router.get("/")
async def list_tasks(): ...

@tasks_router.get("/{task_id}")
async def get_task(task_id: int): ...


# src/api/v1/__init__.py
from fastapi import APIRouter
from .tasks import tasks_router
from .categories import categories_router

# The v1 "umbrella" router — sets the version prefix once
v1_router = APIRouter(prefix="/v1")
v1_router.include_router(tasks_router,      prefix="/tasks",      tags=["Tasks"])
v1_router.include_router(categories_router, prefix="/categories", tags=["Categories"])


# src/api/v2/__init__.py
from fastapi import APIRouter
from .tasks import tasks_v2_router      # New v2 task router (different response shape)
from ..v1.categories import categories_router  # Categories unchanged — REUSE the v1 router

v2_router = APIRouter(prefix="/v2")
v2_router.include_router(tasks_v2_router,   prefix="/tasks",      tags=["Tasks"])
v2_router.include_router(categories_router, prefix="/categories", tags=["Categories"])
#                         ↑ same categories router included under /v2 with zero duplication


# src/main.py
from fastapi import FastAPI
from src.api.v1 import v1_router
from src.api.v2 import v2_router

app = FastAPI()
app.include_router(v1_router)   # Registers: /v1/tasks/..., /v1/categories/...
app.include_router(v2_router)   # Registers: /v2/tasks/..., /v2/categories/...
```

```
┌─────────────────────────────────────────────────────────────────┐
│              NESTED ROUTER BENEFIT                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  If categories don't change between v1 and v2, you include      │
│  the SAME categories_router in BOTH version routers.            │
│  No code duplication. One router, two URL prefixes.             │
│                                                                 │
│  v1_router.include_router(categories_router, prefix="/categories")│
│  v2_router.include_router(categories_router, prefix="/categories")│
│                                                                 │
│  Result:                                                        │
│  GET /v1/categories  → handled by categories_router             │
│  GET /v2/categories  → handled by categories_router             │
│  (same code, same logic, different URL)                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The `responses` parameter — shared response schemas in OpenAPI:**

```python
from src.api.errors import ProblemDetail   # You'll define this in Part 7

v1_router.include_router(
    tasks_router,
    prefix="/tasks",
    tags=["Tasks"],
    responses={
        # These appear in the OpenAPI docs for EVERY route in tasks_router
        # without repeating them on each individual endpoint
        401: {
            "model": ProblemDetail,
            "description": "Not authenticated — provide a valid token.",
        },
        403: {
            "model": ProblemDetail,
            "description": "Authenticated but not authorised for this resource.",
        },
        404: {
            "model": ProblemDetail,
            "description": "Task not found.",
        },
    },
)
```

> The `responses` parameter does NOT add any runtime behaviour. It is purely a documentation signal to OpenAPI. Your `exception_handler` still handles the actual responses. Think of `responses` as "telling the spec what is already true about your handlers."

```
┌─────────────────────────────────────────────────────────────────┐
│                    ✏️  PRACTICE CHECKPOINT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q: You have this setup:                                        │
│                                                                 │
│     tasks = APIRouter(prefix="/tasks")                          │
│                                                                 │
│     @tasks.get("/{task_id}/tags")                               │
│     async def get_task_tags(): ...                              │
│                                                                 │
│     v1 = APIRouter(prefix="/v1")                                │
│     v1.include_router(tasks, prefix="/api")                     │
│     app.include_router(v1)                                      │
│                                                                 │
│     What is the final URL of get_task_tags?                     │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  SOLUTION                                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  /v1/api/tasks/{task_id}/tags                                   │
│                                                                 │
│  Prefix stacking is additive left-to-right:                     │
│   app.include_router(v1)       → adds /v1                       │
│   v1.include_router(tasks, prefix="/api") → adds /api           │
│   APIRouter(prefix="/tasks")   → adds /tasks                    │
│   @tasks.get("/{task_id}/tags") → adds /{task_id}/tags          │
│                                                                 │
│  Combined: /v1 + /api + /tasks + /{task_id}/tags                │
│         = /v1/api/tasks/{task_id}/tags                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Input/Output Model Separation (The Mass Assignment Problem)

**The bug you don't know you have:**

```python
# ❌ VULNERABLE: one model for everything
class Task(BaseModel):
    id: int
    title: str
    status: str
    owner_id: int      # Who owns this task
    created_at: str    # When it was created
    is_admin: bool     # Whether the task owner is an admin

@router.post("/tasks")
async def create_task(task: Task):   # Accepts the FULL model as input
    ALL_TASKS.append(task.model_dump())
    return task
```

**What happens when a malicious client sends this:**

```json
POST /tasks
{
    "title": "Legitimate task",
    "status": "pending",
    "owner_id": 999,
    "created_at": "1970-01-01",
    "is_admin": true
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE MASS ASSIGNMENT ATTACK                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The client submitted owner_id and is_admin.                    │
│  Your API accepted them. The model validated them.              │
│  They are now in your storage exactly as sent.                  │
│                                                                 │
│  The attacker just:                                             │
│  ├─ Assigned the task to a different user (owner_id: 999)       │
│  ├─ Backdated it to the Unix epoch (created_at: 1970-01-01)     │
│  └─ Granted themselves admin (is_admin: true)                   │
│                                                                 │
│  Pydantic validated the types. It did exactly what you told     │
│  it to do. The vulnerability is in the DESIGN, not the code.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The fix: three models, three responsibilities**

```python
from datetime import datetime
from pydantic import BaseModel, Field


# ─── INPUT: What the client is ALLOWED to send on creation ───────

class TaskCreate(BaseModel):
    """Fields accepted when creating a task. Nothing else."""
    title: str = Field(..., min_length=1, max_length=200)
    description: str | None = None
    priority: int = Field(default=0, ge=0, le=10)
    # Deliberately absent:
    # id         → auto-assigned by server
    # owner_id   → set from authenticated user, never from client
    # created_at → set by server at creation time
    # status     → always "pending" on creation; client cannot override


# ─── INPUT: What the client is ALLOWED to send on update ─────────

class TaskUpdate(BaseModel):
    """Fields accepted in a PATCH request. All optional — send only what changes."""
    title: str | None = Field(default=None, min_length=1, max_length=200)
    description: str | None = None
    priority: int | None = Field(default=None, ge=0, le=10)
    status: str | None = None
    # Still absent:
    # id         → immutable forever
    # owner_id   → you cannot reassign ownership via PATCH
    # created_at → immutable forever


# ─── OUTPUT: What the API RETURNS to the client ───────────────────

class TaskResponse(BaseModel):
    """The shape of a task in every API response. Read-only from client's view."""
    id: int
    title: str
    description: str | None
    status: str
    priority: int
    owner_id: int           # Client can READ this, but never set it via input
    created_at: datetime    # Client can READ this, but never set it via input
    updated_at: datetime


# ─── USAGE ────────────────────────────────────────────────────────

@router.post("", status_code=201, response_model=TaskResponse)
async def create_task(task: TaskCreate):     # ← Input model: only allowed fields
    new_task = {
        "id": generate_id(),
        "title": task.title,
        "description": task.description,
        "priority": task.priority,
        "status": "pending",                 # ← Server-enforced, not client-set
        "owner_id": 1,                       # ← From auth (stub; Week 9 replaces this)
        "created_at": datetime.utcnow(),     # ← Server-enforced
        "updated_at": datetime.utcnow(),
    }
    ALL_TASKS.append(new_task)
    return new_task                          # ← Serialised through TaskResponse
```

```
┌─────────────────────────────────────────────────────────────────┐
│               THE THREE-MODEL RULE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TaskCreate   → "What can you send me to CREATE a task?"        │
│  TaskUpdate   → "What can you send me to CHANGE a task?"        │
│  TaskResponse → "What will I always send YOU about a task?"     │
│                                                                 │
│  They are NOT the same. Fields that appear in TaskResponse      │
│  do NOT automatically appear in TaskCreate or TaskUpdate.       │
│                                                                 │
│  ASK YOURSELF FOR EVERY FIELD:                                  │
│  ├─ Can the client set this on creation?   → TaskCreate         │
│  ├─ Can the client change this after?      → TaskUpdate         │
│  ├─ Should the client be able to READ it?  → TaskResponse       │
│  └─ Is it server-only (id, timestamps)?    → NONE of the above  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice that `response_model=TaskResponse` on the endpoint acts as a second line of defence. Even if your handler accidentally returns extra fields, Pydantic strips them before the response leaves the server. But this is not an excuse — the input models are the actual security boundary."

```
┌─────────────────────────────────────────────────────────────────┐
│                    ✏️  PRACTICE CHECKPOINT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q: A client sends this request:                                │
│                                                                 │
│     POST /tasks                                                 │
│     {"title": "Write docs", "owner_id": 42, "created_at":      │
│      "2000-01-01T00:00:00"}                                     │
│                                                                 │
│     Your endpoint uses TaskCreate as its input model.           │
│     TaskCreate has only: title, description, priority.          │
│                                                                 │
│     What happens to owner_id and created_at?                    │
│     Does Pydantic raise an error?                               │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  SOLUTION                                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  By default, Pydantic IGNORES extra fields — no error is        │
│  raised. owner_id and created_at simply do not appear in        │
│  the parsed TaskCreate object. They are silently discarded.     │
│                                                                 │
│  This is the safe behaviour you want. The attacker gets no      │
│  error feedback, and their fields never reach your handler.     │
│                                                                 │
│  (To make extra fields raise a 422 instead of being ignored:    │
│   model_config = ConfigDict(extra="forbid") on TaskCreate.)     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

---

# PART 3: PAGINATION

## 3.1 The "Dump Everything" Problem

**Back to our terrible API. Let's measure the damage.**

```python
# What happens when GET /tasks returns everything?

# 100 tasks     → ~15 KB    → Fine
# 1,000 tasks   → ~150 KB   → Slowish on mobile
# 10,000 tasks  → ~1.5 MB   → Painful
# 100,000 tasks → ~15 MB    → Unusable
# 1,000,000     → ~150 MB   → Server dies. Client dies. You die.
```

```
┌─────────────────────────────────────────────────────────────────┐
│                THE "DUMP EVERYTHING" PROBLEM                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT REQUESTS:  GET /tasks                                   │
│  CLIENT NEEDS:     The 20 most recent tasks                     │
│  SERVER RETURNS:   ALL 50,000 tasks                             │
│                                                                 │
│  WHAT GETS WASTED:                                              │
│  ├─ Server memory: loads 50,000 objects into RAM                │
│  ├─ Server CPU: serializes 50,000 objects to JSON               │
│  ├─ Bandwidth: sends 4.2 MB over the network                   │
│  ├─ Client memory: parses 50,000 objects from JSON              │
│  ├─ Client CPU: renders... then crashes                         │
│  └─ User's patience: gone                                       │
│                                                                 │
│                                                                 │
│  STORE ANALOGY:                                                 │
│  ──────────────                                                 │
│  Imagine searching for shoes on Amazon and instead of           │
│  showing 20 results per page, it dumps ALL 500,000 shoes        │
│  onto one page. Your browser would melt.                        │
│                                                                 │
│  No online store does this. Your API shouldn't either.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The solution: pagination.**

> "Don't give them the whole warehouse. Give them one page of the catalog at a time, with a way to turn to the next page."

---

## 3.2 Offset-Based Pagination (The Simple Way)

**The simplest pagination. You say: "Give me 20 items, starting from item 40."**

```
GET /tasks?limit=20&offset=0     ← Page 1 (items 0-19)
GET /tasks?limit=20&offset=20    ← Page 2 (items 20-39)
GET /tasks?limit=20&offset=40    ← Page 3 (items 40-59)
```

**Visualize it:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 OFFSET-BASED PAGINATION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your data:                                                     │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐     │
│  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │10 │11 │12 │...│   │
│  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘     │
│                                                                 │
│  limit=5, offset=0:     [0][1][2][3][4]                         │
│                          ▲                                      │
│                          └─ Start here, take 5                  │
│                                                                 │
│  limit=5, offset=5:                    [5][6][7][8][9]          │
│                                         ▲                       │
│                                         └─ Skip 5, take 5      │
│                                                                 │
│  limit=5, offset=10:                              [10][11][12]… │
│                                                    ▲            │
│                                                    └─ Skip 10   │
│                                                                 │
│  FORMULA:                                                       │
│  Page N (0-indexed) = offset: N × limit, limit: page_size      │
│  Page 0 → offset=0,  limit=20                                  │
│  Page 1 → offset=20, limit=20                                  │
│  Page 2 → offset=40, limit=20                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation in FastAPI:**

```python
from fastapi import APIRouter, Query

router = APIRouter(prefix="/v1/tasks", tags=["tasks"])

# In-memory storage for now (database comes in Week 5)
ALL_TASKS: list[dict] = [...]  # Imagine 500 tasks here


@router.get("")
async def get_tasks(
    limit: int = Query(default=20, ge=1, le=100),  # Enforce bounds!
    offset: int = Query(default=0, ge=0),
):
    # Slice the data
    tasks = ALL_TASKS[offset : offset + limit]

    return {
        "items": tasks,
        "total": len(ALL_TASKS),
        "limit": limit,
        "offset": offset,
    }
```

**Connection to Pydantic (Week 3 Lecture 3):**

> "Remember `Field` constraints from Pydantic? `Query(ge=1, le=100)` uses the same validation idea. We cap `limit` at 100 so no one can request `?limit=999999` and bypass our pagination."

**The response:**

```json
{
    "items": [
        {"id": 41, "title": "Task 41", "status": "pending"},
        {"id": 42, "title": "Task 42", "status": "done"}
    ],
    "total": 500,
    "limit": 20,
    "offset": 40
}
```

> "The `total` field tells the client how many items exist in total. Combined with `limit`, the client can calculate how many pages there are: `total_pages = ceil(total / limit)`. This is how the frontend renders those page number buttons."

**Offset pagination seems perfect. It's simple, intuitive, and easy to implement.**

**Now let me show you why it breaks.**

---

## 3.3 The Shifting Window Trap

**The problem: data changes between page requests.**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE SHIFTING WINDOW PROBLEM                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: Client requests page 1                                 │
│                                                                 │
│  Data (sorted by newest first):                                 │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐            │
│  │ T1 │ T2 │ T3 │ T4 │ T5 │ T6 │ T7 │ T8 │ T9 │T10│           │
│  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘            │
│  [========PAGE 1=========]                                      │
│   offset=0, limit=5                                             │
│                                                                 │
│  Client gets: T1, T2, T3, T4, T5  ✅                            │
│                                                                 │
│                                                                 │
│  STEP 2: Someone creates a NEW task (T0) before client          │
│          requests page 2                                        │
│                                                                 │
│  Data is NOW:                                                   │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐       │
│  │ T0 │ T1 │ T2 │ T3 │ T4 │ T5 │ T6 │ T7 │ T8 │ T9 │T10│      │
│  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘       │
│                          [========PAGE 2=========]              │
│                           offset=5, limit=5                     │
│                                                                 │
│  Client gets: T5, T6, T7, T8, T9                               │
│               ──                                                │
│               └─ T5 AGAIN! Client already saw T5 on page 1!    │
│                                                                 │
│  RESULT: T5 appears TWICE. T0 was never seen. 💥                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The reverse problem — deletion:**

```
┌─────────────────────────────────────────────────────────────────┐
│             SHIFTING WINDOW: DELETION                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Page 1 requested:                                              │
│  [T1][T2][T3][T4][T5] [T6][T7][T8][T9][T10]                    │
│  [====PAGE 1=====]                                              │
│                                                                 │
│  Client gets: T1, T2, T3, T4, T5                               │
│                                                                 │
│  Then T3 gets DELETED:                                          │
│  [T1][T2][T4][T5][T6] [T7][T8][T9][T10]                        │
│                        [====PAGE 2====]                         │
│                         offset=5                                │
│                                                                 │
│  Client gets: T7, T8, T9, T10                                  │
│                                                                 │
│  RESULT: T6 was SKIPPED. Never shown. 💥                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "So offset pagination is fundamentally broken? Should we never use it?"

Answer:

```
┌─────────────────────────────────────────────────────────────────┐
│            OFFSET PAGINATION: THE HONEST TRUTH                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The shifting window problem is REAL, but it matters            │
│  more in some cases than others:                                │
│                                                                 │
│  OFFSET IS FINE WHEN:                                           │
│  ├─ Data changes infrequently                                   │
│  ├─ Exact consistency doesn't matter (browsing blog posts)      │
│  ├─ You need "jump to page 47" functionality                    │
│  └─ The dataset is small-to-medium                              │
│                                                                 │
│  OFFSET HURTS WHEN:                                             │
│  ├─ Data changes frequently (real-time feeds, active tasks)     │
│  ├─ Missing or duplicating items is unacceptable                │
│  ├─ Deep pages (OFFSET 50000 is SLOW in databases)              │
│  └─ Infinite scroll UI (user scrolls through a stream)          │
│                                                                 │
│  THERE'S ALSO A PERFORMANCE PROBLEM:                            │
│  In a database, OFFSET 50000 means:                             │
│  "Scan 50,000 rows, throw them away, return the next 20."      │
│  The deeper the page, the slower the query.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Cursor-Based Pagination (The Robust Way)

**Instead of saying "skip N items," we say "give me items AFTER this marker."**

```
┌─────────────────────────────────────────────────────────────────┐
│                 CURSOR-BASED PAGINATION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OFFSET says: "Skip 40 items, give me the next 20."            │
│  CURSOR says: "Give me 20 items AFTER task #40."                │
│                                                                 │
│  The cursor is a BOOKMARK — it points to the last item          │
│  you saw, not a position number.                                │
│                                                                 │
│                                                                 │
│  STORE ANALOGY:                                                 │
│  ──────────────                                                 │
│  OFFSET: "Show me page 3 of shoes."                             │
│          (If new shoes arrive, page 3 shifts.)                  │
│                                                                 │
│  CURSOR: "Show me shoes listed AFTER the red Nike Air Max."     │
│          (New arrivals don't affect what comes AFTER a          │
│           specific shoe. Your place is stable.)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How it works:**

```
┌─────────────────────────────────────────────────────────────────┐
│               CURSOR PAGINATION FLOW                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REQUEST 1: GET /tasks?limit=5                                  │
│  (no cursor = start from the beginning)                         │
│                                                                 │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐            │
│  │ T1 │ T2 │ T3 │ T4 │ T5 │ T6 │ T7 │ T8 │ T9 │T10│           │
│  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘            │
│  [========RETURNED========]  ▲                                  │
│                              └─ cursor now points to T5         │
│                                                                 │
│  RESPONSE: { items: [T1-T5], next_cursor: "cursor_for_T5" }    │
│                                                                 │
│                                                                 │
│  REQUEST 2: GET /tasks?limit=5&cursor=cursor_for_T5             │
│  (give me items AFTER T5)                                       │
│                                                                 │
│  Even if T0 was inserted:                                       │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐       │
│  │ T0 │ T1 │ T2 │ T3 │ T4 │ T5 │ T6 │ T7 │ T8 │ T9 │T10│      │
│  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘       │
│                              ▲  [========RETURNED========]      │
│                              └─ cursor: "after T5"              │
│                                                                 │
│  RESPONSE: { items: [T6-T10], next_cursor: "cursor_for_T10" }  │
│                                                                 │
│  NO DUPLICATES. NO SKIPS. T0 didn't affect our position. ✅     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The cursor is typically an encoded reference to the last item seen:**

```python
import base64

# The cursor is opaque to the client — they don't know what's inside.
# Internally, it's often an encoded ID or timestamp.

def encode_cursor(task_id: int) -> str:
    """Encode a task ID into an opaque cursor string."""
    raw = f"id:{task_id}".encode()
    return base64.urlsafe_b64encode(raw).decode()
    # "id:42" → "aWQ6NDI="


def decode_cursor(cursor: str) -> int:
    """Decode cursor back to a task ID."""
    raw = base64.urlsafe_b64decode(cursor.encode()).decode()
    # "aWQ6NDI=" → "id:42"
    prefix, value = raw.split(":")
    return int(value)
```

**Why encode it?**

> "The cursor should be **opaque** to the client. They treat it as a magic token. They don't parse it, don't construct it, don't depend on its format. This lets you change the internal encoding anytime without breaking clients."

**Implementation in FastAPI:**

```python
from fastapi import APIRouter, Query
from pydantic import BaseModel

router = APIRouter(prefix="/v1/tasks", tags=["tasks"])


class PaginatedResponse(BaseModel):
    items: list[dict]
    next_cursor: str | None    # None means "no more pages"
    has_more: bool


@router.get("", response_model=PaginatedResponse)
async def get_tasks(
    limit: int = Query(default=20, ge=1, le=100),
    cursor: str | None = Query(default=None),
):
    # Determine starting point
    if cursor is None:
        start_index = 0
    else:
        last_seen_id = decode_cursor(cursor)
        # Find position of last seen item
        start_index = next(
            (i + 1 for i, t in enumerate(ALL_TASKS) if t["id"] == last_seen_id),
            0,
        )

    # Slice the data
    tasks = ALL_TASKS[start_index : start_index + limit]

    # Build next cursor
    has_more = start_index + limit < len(ALL_TASKS)
    next_cursor = encode_cursor(tasks[-1]["id"]) if tasks and has_more else None

    return PaginatedResponse(
        items=tasks,
        next_cursor=next_cursor,
        has_more=has_more,
    )
```

**Client usage:**

```python
# Client code (pseudocode):
page1 = GET("/tasks?limit=20")
# Use page1.items...

if page1.has_more:
    page2 = GET(f"/tasks?limit=20&cursor={page1.next_cursor}")
    # Use page2.items...

    if page2.has_more:
        page3 = GET(f"/tasks?limit=20&cursor={page2.next_cursor}")
        # ...and so on
```

---

## 3.5 Offset vs Cursor: The Decision

```
┌─────────────────────────────────────────────────────────────────┐
│              OFFSET VS CURSOR: DECISION GUIDE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                │ Offset              │ Cursor                   │
│  ──────────────┼─────────────────────┼────────────────────────  │
│  Jump to page  │ ✅ Easy             │ ❌ Cannot                │
│  "Page 47"     │ offset = 46 × limit │ Must traverse from start │
│                │                     │                          │
│  Deep pages    │ ❌ Gets slower      │ ✅ Constant speed        │
│  (page 500)    │ DB scans all rows   │ Uses index, always fast  │
│                │                     │                          │
│  Data changes  │ ❌ Shifts/duplicates│ ✅ Stable position       │
│  between pages │ Items skipped/duped │ Cursor holds its place   │
│                │                     │                          │
│  Total count   │ ✅ Easy to provide  │ ⚠️ Expensive to compute  │
│                │ SELECT COUNT(*)     │ Must scan all rows       │
│                │                     │                          │
│  Simplicity    │ ✅ Very simple      │ ⚠️ More implementation   │
│                │ Just slice          │ Encoding, decoding       │
│                │                     │                          │
│  Client logic  │ ✅ Simple math      │ ✅ Follow next_cursor    │
│                │ offset += limit     │ Just pass the token      │
│                │                     │                          │
│  Best for      │ Admin dashboards,   │ Feeds, timelines,        │
│                │ small datasets,     │ large datasets, mobile   │
│                │ "jump to page"      │ infinite scroll          │
│                │                     │                          │
│  ──────────────┼─────────────────────┼────────────────────────  │
│  Used by       │ Most admin panels   │ Twitter, Slack,          │
│                │ Traditional web     │ Facebook, Stripe         │
│                │                     │                          │
└─────────────────────────────────────────────────────────────────┘
```

**For your Task Manager project:**

> "Your project requires **cursor-based pagination**. Here's why: tasks are created and deleted frequently. Offset pagination would give inconsistent results. Cursor pagination guarantees stable page traversal. This is the pattern used by most production APIs."

---

## 3.6 Response Envelope Design (Connection to Pydantic)

**Connection to what you've learned:**

> "Remember Pydantic's `BaseModel` and nested models from Week 3 Lecture 3? Now we use them to define a **standard response envelope** — a consistent wrapper around all paginated responses."

**Don't do this — inconsistent responses:**

```python
# ❌ BAD: Every endpoint has a different shape
GET /tasks   → [{"id": 1, ...}, {"id": 2, ...}]          # bare list
GET /users   → {"data": [...], "count": 50}               # "data" key
GET /tags    → {"results": [...], "total": 10, "pg": 1}   # "results"? "pg"?
```

**Do this — a consistent envelope:**

```python
# ✅ GOOD: Every paginated endpoint uses the same shape

from pydantic import BaseModel
from typing import Generic, TypeVar

T = TypeVar("T")


class PaginationMeta(BaseModel):
    """Standard pagination metadata."""
    next_cursor: str | None
    has_more: bool
    # For offset pagination you could add:
    # total: int | None = None
    # offset: int | None = None


class PaginatedResponse(BaseModel, Generic[T]):
    """Standard envelope for all paginated endpoints."""
    items: list[T]
    pagination: PaginationMeta
```

**Connection to Generics (Week 1 Lecture 1):**

> "Remember `TypeVar` from Lecture 1? Here it becomes genuinely useful. `PaginatedResponse[TaskResponse]` means 'a paginated response where each item is a `TaskResponse`.' One envelope model works for tasks, categories, tags — everything."

```python
# Usage:

class TaskResponse(BaseModel):
    id: int
    title: str
    status: str


class CategoryResponse(BaseModel):
    id: int
    name: str


# Same envelope, different item types:
# PaginatedResponse[TaskResponse]
# PaginatedResponse[CategoryResponse]


@router.get("/tasks", response_model=PaginatedResponse[TaskResponse])
async def get_tasks(
    limit: int = Query(default=20, ge=1, le=100),
    cursor: str | None = Query(default=None),
):
    tasks, next_cursor, has_more = fetch_tasks(limit, cursor)

    return PaginatedResponse(
        items=tasks,
        pagination=PaginationMeta(
            next_cursor=next_cursor,
            has_more=has_more,
        ),
    )
```

**What the client sees — always predictable:**

```json
{
    "items": [
        {"id": 1, "title": "Write tests", "status": "pending"},
        {"id": 2, "title": "Code review", "status": "done"}
    ],
    "pagination": {
        "next_cursor": "aWQ6Mg==",
        "has_more": true
    }
}
```

> "Every paginated endpoint returns `items` and `pagination`. The client team writes their pagination logic ONCE and it works for every resource. That's the power of a consistent envelope."

---

# PART 4: FILTERING AND SORTING

## 4.1 The Client Shouldn't Do Your Job

**Scenario: the frontend needs "all pending tasks, sorted by newest first."**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE FILTERING PROBLEM                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT SERVER-SIDE FILTERING:                                 │
│  ──────────────────────────────                                 │
│                                                                 │
│  Client: GET /tasks?limit=20                                    │
│  Server: here are 20 tasks (mixed statuses)                     │
│  Client: I only want "pending" ones... got 7 out of 20.        │
│  Client: GET /tasks?limit=20&offset=20                          │
│  Server: here are 20 more                                       │
│  Client: 5 pending here. Still don't have 20 pending tasks.    │
│  Client: GET /tasks?limit=20&offset=40 ...                     │
│                                                                 │
│  The client fetches PAGES AND PAGES just to find 20             │
│  pending tasks. Wasted bandwidth. Slow. Awful.                 │
│                                                                 │
│                                                                 │
│  WITH SERVER-SIDE FILTERING:                                    │
│  ───────────────────────────                                    │
│                                                                 │
│  Client: GET /tasks?status=pending&limit=20                     │
│  Server: here are 20 pending tasks.                             │
│  Client: 👌                                                     │
│                                                                 │
│  One request. Exact data. Done.                                 │
│                                                                 │
│                                                                 │
│  STORE ANALOGY:                                                 │
│  ──────────────                                                 │
│  Without filtering: "Show me ALL products. I'll look through   │
│                      thousands to find the red shoes myself."   │
│                                                                 │
│  With filtering:    "Show me red shoes, size 10, under $100."  │
│                      Three clicks. Exactly what you need.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Filtering Conventions

**Filters go in query parameters. The conventions are simple:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  FILTERING CONVENTIONS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EXACT MATCH:                                                   │
│  GET /tasks?status=pending                                      │
│  GET /tasks?priority=3                                          │
│  GET /tasks?category_id=7                                       │
│                                                                 │
│  MULTIPLE VALUES (OR logic):                                    │
│  GET /tasks?status=pending&status=in_progress                   │
│  or                                                             │
│  GET /tasks?status=pending,in_progress                          │
│                                                                 │
│  SEARCH (partial text match):                                   │
│  GET /tasks?search=deploy                                       │
│  (matches "Deploy to production", "Fix deploy script", etc.)    │
│                                                                 │
│  DATE RANGES:                                                   │
│  GET /tasks?created_after=2025-01-01&created_before=2025-02-01  │
│                                                                 │
│                                                                 │
│  NAMING CONVENTIONS:                                            │
│  ├─ Use the same field names as your response body              │
│  │   Response has "status"? Filter with ?status=                │
│  │   NOT ?task_status= or ?s= or ?filter_status=               │
│  │                                                              │
│  ├─ Use snake_case (Python convention, you already do this)     │
│  │                                                              │
│  └─ Suffix for ranges: _after, _before, _min, _max, _gte, _lte│
│     ?price_min=10&price_max=50                                  │
│     ?created_after=2025-01-01                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation in FastAPI:**

```python
@router.get("/tasks", response_model=PaginatedResponse[TaskResponse])
async def get_tasks(
    # Pagination
    limit: int = Query(default=20, ge=1, le=100),
    cursor: str | None = Query(default=None),
    # Filters
    status: str | None = Query(default=None),
    priority: int | None = Query(default=None),
    category_id: int | None = Query(default=None),
    search: str | None = Query(default=None),
):
    tasks = ALL_TASKS

    # Apply filters (only if the parameter was provided)
    if status is not None:
        tasks = [t for t in tasks if t["status"] == status]
    if priority is not None:
        tasks = [t for t in tasks if t["priority"] == priority]
    if category_id is not None:
        tasks = [t for t in tasks if t["category_id"] == category_id]
    if search is not None:
        tasks = [t for t in tasks if search.lower() in t["title"].lower()]

    # Then paginate the filtered results
    # ... (pagination logic here) ...
```

**Notice: filtering happens BEFORE pagination.** This is critical.

```
┌─────────────────────────────────────────────────────────────────┐
│              ORDER OF OPERATIONS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  All data (50,000 tasks)                                        │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │  1. FILTER   │  → 2,000 pending tasks remain                 │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │  2. SORT     │  → 2,000 tasks now sorted by created_at      │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │  3. PAGINATE │  → 20 tasks returned to client                │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  Response: 20 pending tasks, newest first, page 1 of 100       │
│                                                                 │
│  FILTER first, SORT second, PAGINATE last. Always.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Sorting Conventions

**Sorting also goes in query parameters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SORTING CONVENTIONS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  APPROACH 1: Two parameters                                     │
│  GET /tasks?sort_by=created_at&order=desc                       │
│  GET /tasks?sort_by=title&order=asc                             │
│                                                                 │
│  APPROACH 2: Single parameter with prefix                       │
│  GET /tasks?sort=-created_at       (- means descending)         │
│  GET /tasks?sort=title             (no prefix means ascending)  │
│                                                                 │
│  APPROACH 3: Single parameter with multiple fields              │
│  GET /tasks?sort=-created_at,title                              │
│  (Sort by created_at DESC, then by title ASC as tiebreaker)     │
│                                                                 │
│                                                                 │
│  FOR YOUR PROJECT (keep it simple):                             │
│  Use Approach 1. It's the most readable and explicit.           │
│  Two parameters. No ambiguity.                                  │
│                                                                 │
│  IMPORTANT: Always have a DEFAULT sort.                         │
│  If the client doesn't specify, sort by something               │
│  deterministic (e.g., created_at DESC, then id DESC).           │
│  Never return unsorted data — the order would be                │
│  unpredictable and pagination would break.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation with validation:**

```python
from enum import Enum


class SortField(str, Enum):
    created_at = "created_at"
    title = "title"
    priority = "priority"
    status = "status"


class SortOrder(str, Enum):
    asc = "asc"
    desc = "desc"


@router.get("/tasks")
async def get_tasks(
    # ... pagination and filter params ...
    sort_by: SortField = Query(default=SortField.created_at),
    order: SortOrder = Query(default=SortOrder.desc),
):
    tasks = ALL_TASKS
    # ... apply filters ...

    # Apply sorting
    reverse = order == SortOrder.desc
    tasks = sorted(tasks, key=lambda t: t[sort_by.value], reverse=reverse)

    # ... apply pagination ...
```

**Why use an Enum?**

> "If the client sends `?sort_by=password` or `?sort_by=; DROP TABLE tasks`, we have a problem. The `Enum` constrains the allowed values. FastAPI returns a 422 validation error automatically if the value isn't in the enum — Pydantic validates it for free."

**Connection to Error Handling (Week 3 Lecture 4):**

> "Remember HTTPException? You don't even need it here. Pydantic + FastAPI's query parameter validation rejects bad sort values before your code ever runs. Validation as security — you saw this in Pydantic Lecture."

---

## 4.4 Combining Everything (Connection to Depends)

**The endpoint signature is getting long. Let's clean it up.**

**Connection to what you've learned:**

> "Remember `Depends()` from Week 3 Lecture 4? You used it for dependency injection. Now we use it to extract reusable parameter groups."

**Before — messy endpoint with 8+ parameters:**

```python
# ❌ Every paginated/filterable endpoint repeats these params
@router.get("/tasks")
async def get_tasks(
    limit: int = Query(default=20, ge=1, le=100),
    cursor: str | None = Query(default=None),
    status: str | None = Query(default=None),
    priority: int | None = Query(default=None),
    category_id: int | None = Query(default=None),
    search: str | None = Query(default=None),
    sort_by: SortField = Query(default=SortField.created_at),
    order: SortOrder = Query(default=SortOrder.desc),
):
    ...

# Now imagine copy-pasting limit+cursor+sort_by+order
# into GET /categories, GET /tags, GET /users...
# Duplication everywhere!
```

**After — clean with Depends:**

```python
from dataclasses import dataclass
from fastapi import Depends


@dataclass
class PaginationParams:
    """Reusable pagination parameters."""
    limit: int = Query(default=20, ge=1, le=100)
    cursor: str | None = Query(default=None)


@dataclass
class TaskFilterParams:
    """Task-specific filter parameters."""
    status: str | None = Query(default=None)
    priority: int | None = Query(default=None)
    category_id: int | None = Query(default=None)
    search: str | None = Query(default=None)


@dataclass
class SortParams:
    """Reusable sort parameters."""
    sort_by: SortField = Query(default=SortField.created_at)
    order: SortOrder = Query(default=SortOrder.desc)


# NOW: clean endpoint signatures
@router.get("/tasks", response_model=PaginatedResponse[TaskResponse])
async def get_tasks(
    pagination: PaginationParams = Depends(),
    filters: TaskFilterParams = Depends(),
    sorting: SortParams = Depends(),
):
    tasks = ALL_TASKS

    # 1. Filter
    if filters.status:
        tasks = [t for t in tasks if t["status"] == filters.status]
    if filters.priority:
        tasks = [t for t in tasks if t["priority"] == filters.priority]
    # ... more filters ...

    # 2. Sort
    reverse = sorting.order == SortOrder.desc
    tasks = sorted(tasks, key=lambda t: t[sorting.sort_by.value], reverse=reverse)

    # 3. Paginate
    start = resolve_cursor(pagination.cursor)
    page = tasks[start : start + pagination.limit]
    # ... build response ...
```

**Connection to Dataclasses (Week 1 Lecture 2):**

> "Remember `@dataclass` from the first week? We said it was a 'foundation for Pydantic understanding.' Here it shows up again — `PaginationParams` and `SortParams` are dataclasses that FastAPI knows how to inject. The `@dataclass` decorator makes them work with `Depends()` seamlessly."

**Now `PaginationParams` and `SortParams` can be reused across every endpoint:**

```python
@router.get("/categories", response_model=PaginatedResponse[CategoryResponse])
async def get_categories(
    pagination: PaginationParams = Depends(),  # Reused!
    sorting: SortParams = Depends(),           # Reused!
):
    ...

@router.get("/tags", response_model=PaginatedResponse[TagResponse])
async def get_tags(
    pagination: PaginationParams = Depends(),  # Reused!
    sorting: SortParams = Depends(),           # Reused!
):
    ...
```

> "Write it once. Use it everywhere. That's the payoff of dependency injection."


## 4.5 Nested Resources vs Query Parameters

**Connection to what you've learned:**

> "You just learned `GET /tasks?category_id=7` for filtering. But sometimes the right URL is `GET /tasks/42/tags` instead. How do you decide? This is one of the most common REST design questions in backend interviews — and real API reviews."

**The two patterns, side by side:**

```
┌─────────────────────────────────────────────────────────────────┐
│            NESTED RESOURCE vs QUERY PARAMETER                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NESTED RESOURCE:                                               │
│  GET /tasks/42/comments                                         │
│       ──────────                                                │
│       "Comments that belong to task 42"                         │
│       Parent (task) is part of the URL path.                   │
│                                                                 │
│  QUERY PARAMETER:                                               │
│  GET /comments?task_id=42                                       │
│                ──────────                                       │
│       "Comments, filtered to those with task_id 42"            │
│       Parent is a filter on a flat resource.                    │
│                                                                 │
│  SAME data. Radically different design message.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The decision framework:**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE NESTING DECISION FRAMEWORK                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Ask these three questions:                                     │
│                                                                 │
│  1. Can the child resource exist WITHOUT the parent?            │
│     YES → Query parameter. (tags exist independently)           │
│     NO  → Nested resource. (comments only exist on a task)      │
│                                                                 │
│  2. Does accessing the child ALWAYS require the parent ID?      │
│     YES → Nested resource. (you always need task ID for         │
│            a task's comments; you never browse all comments)    │
│     NO  → Query parameter. (you sometimes browse all tags,      │
│            sometimes filter by task)                            │
│                                                                 │
│  3. Is the relationship one-to-one from parent to child?        │
│     YES → Nested makes intuitive sense.                         │
│            GET /tasks/42/status                                 │
│     NO  → Use query params for flexibility.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Applied to your Task Manager:**

```
┌─────────────────────────────────────────────────────────────────┐
│            TASK MANAGER: NESTING DECISIONS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NESTED (child cannot exist without parent):                    │
│  GET    /tasks/{task_id}/comments          ← comments of a task │
│  POST   /tasks/{task_id}/comments          ← add comment        │
│  DELETE /tasks/{task_id}/comments/{id}     ← remove comment     │
│                                                                 │
│  QUERY PARAMETER (child exists independently):                  │
│  GET /tasks?category_id=7    ← tasks filtered by category       │
│  GET /tasks?tag=urgent       ← tasks with a specific tag        │
│  GET /categories             ← categories (unrelated to tasks)  │
│  GET /tags                   ← tags (shared across resources)   │
│                                                                 │
│                                                                 │
│  THE DEPTH RULE: Never go deeper than 2 levels.                 │
│  ────────────────                                               │
│  /tasks/{id}/comments                    ✅ 2 levels            │
│  /tasks/{id}/comments/{id}               ✅ 2 levels            │
│  /tasks/{id}/comments/{id}/likes         ⚠️ 3 levels — stop     │
│                                                                 │
│  At 3+ levels, the URL becomes unwieldy and the hierarchy       │
│  starts lying about the data model. Instead, flatten it:        │
│  GET /likes?comment_id={id}   ← query param rescues the nesting │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation in FastAPI — nested resource with path parameter:**

```python
# Two separate routers: one for tasks, one for comments
tasks_router = APIRouter(prefix="/tasks", tags=["Tasks"])
comments_router = APIRouter(tags=["Comments"])  # No prefix here


# ── TASKS ROUTER ─────────────────────────────────────────────────

@tasks_router.get("/{task_id}", response_model=TaskResponse)
async def get_task(task_id: int):
    task = next((t for t in ALL_TASKS if t["id"] == task_id), None)
    if task is None:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")
    return task


# ── COMMENTS NESTED UNDER TASKS ──────────────────────────────────

@comments_router.get(
    "/tasks/{task_id}/comments",
    response_model=PaginatedResponse[CommentResponse],
)
async def list_task_comments(
    task_id: int,
    pagination: PaginationParams = Depends(),
):
    # Validate the parent exists first — always
    task = next((t for t in ALL_TASKS if t["id"] == task_id), None)
    if task is None:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")

    comments = [c for c in ALL_COMMENTS if c["task_id"] == task_id]
    # ... paginate and return ...


@comments_router.post(
    "/tasks/{task_id}/comments",
    response_model=CommentResponse,
    status_code=201,
)
async def create_comment(task_id: int, body: CommentCreate):
    # Validate the parent task exists before creating the child
    task = next((t for t in ALL_TASKS if t["id"] == task_id), None)
    if task is None:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")

    new_comment = {
        "id": generate_id(),
        "task_id": task_id,          # Bound to parent permanently
        "text": body.text,
        "created_at": datetime.utcnow(),
    }
    ALL_COMMENTS.append(new_comment)
    return new_comment
```

> "Notice: validate the parent exists before touching the child. A `POST /tasks/9999/comments` where task 9999 does not exist should return 404, not 201. Always walk down the hierarchy from parent to child, validating each level."

```
┌─────────────────────────────────────────────────────────────────┐
│                    ✏️  PRACTICE CHECKPOINT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q: A user can belong to many organisations.                    │
│     An organisation has many projects.                          │
│     Each project has many tasks.                                │
│                                                                 │
│     A developer proposes this endpoint:                         │
│     GET /users/42/organisations/7/projects/3/tasks              │
│                                                                 │
│     What is wrong with this URL and how would you fix it?       │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  SOLUTION                                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Problem: 4 levels of nesting. The URL is hostile to read,      │
│  hard to route, and exposes implementation details of the       │
│  data model. It also forces the client to know the user ID      │
│  even though tasks belong to projects, not users.               │
│                                                                 │
│  Fix — flatten where the hierarchy doesn't add clarity:         │
│                                                                 │
│  GET /projects/3/tasks                                          │
│       (2 levels — project → tasks)                              │
│                                                                 │
│  The user context comes from authentication (Part 4.6),         │
│  not from the URL. The org and project IDs validate             │
│  ownership implicitly when you fetch the project from DB.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4.6 [NEW]

## 4.6 Ownership-Aware Filtering

**The problem the "Consumer First" philosophy missed:**

> "We've been designing `GET /tasks` as if it belongs to a single user. It doesn't. In any real Task Manager, `GET /tasks` must answer: *whose* tasks? If you don't answer that question in your design, you've answered it by accident — and the answer is 'everyone's.'"

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE GHOST USER PROBLEM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CURRENT STATE (Part 4.4's implementation):                     │
│                                                                 │
│  User Alice: GET /tasks?status=pending                          │
│  ↳ Returns: ALL pending tasks from ALL users in the system.     │
│     Alice sees Bob's tasks. Bob sees Alice's tasks.             │
│     Every user's data is public to every other user.            │
│                                                                 │
│  This isn't a bug in the filter logic. It's a missing           │
│  dimension in the API design. No consumer of this API           │
│  would expect this behaviour.                                   │
│                                                                 │
│  STORE ANALOGY:                                                  │
│  Imagine clicking "My Orders" on Amazon and seeing every        │
│  order placed by every customer in the world, with your         │
│  filters applied on top. That's what GET /tasks does now.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The architectural fix: ownership is not a filter. It's a precondition.**

Ownership filtering is different from `status=pending` or `sort_by=priority` in one critical way: **the client cannot opt out of it**. It is not in `TaskFilterParams`. It is not a query parameter. It runs first, unconditionally, every time.

```python
# ─── STUB: Week 9 replaces this with real JWT extraction ─────────

async def get_current_user() -> dict:
    """
    Returns the currently authenticated user from the request context.

    In Week 9 this will extract the user from the Authorization header
    (Bearer JWT → decode → look up in DB → return user dict).

    For now: returns a hardcoded user so the ARCHITECTURE is correct
    even before authentication is fully implemented. This means you
    can build, test, and reason about ownership today.
    """
    return {"id": 1, "email": "alice@example.com", "role": "user"}


# ─── UPDATED ENDPOINT: ownership-aware ───────────────────────────

@router.get("", response_model=PaginatedResponse[TaskResponse])
async def get_tasks(
    pagination: PaginationParams = Depends(),
    filters: TaskFilterParams = Depends(),
    sorting: SortParams = Depends(),
    current_user: dict = Depends(get_current_user),  # ← Auth context
):
    tasks = ALL_TASKS

    # ① OWNERSHIP — runs FIRST, ALWAYS, unconditionally
    #   Admin users see everything. Regular users see only their own.
    if current_user["role"] != "admin":
        tasks = [t for t in tasks if t["owner_id"] == current_user["id"]]

    # ② CLIENT-REQUESTED FILTERS (on authorised data only)
    if filters.status is not None:
        tasks = [t for t in tasks if t["status"] == filters.status]
    if filters.priority is not None:
        tasks = [t for t in tasks if t["priority"] == filters.priority]
    if filters.search is not None:
        tasks = [t for t in tasks if filters.search.lower() in t["title"].lower()]

    # ③ SORT
    reverse = sorting.order == SortOrder.desc
    tasks = sorted(tasks, key=lambda t: t[sorting.sort_by.value], reverse=reverse)

    # ④ PAGINATE
    # ... (same cursor pagination logic from Part 3) ...
```

```
┌─────────────────────────────────────────────────────────────────┐
│                ORDER OF OPERATIONS — UPDATED                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  All data (50,000 tasks from all users)                         │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │ 1. OWNERSHIP  │  → User's 500 tasks remain.                  │
│  │   (always)    │    Other users' 49,500 tasks: gone.          │
│  └──────┬────────┘                                              │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │ 2. FILTER     │  → 200 pending tasks remain.                 │
│  └──────┬────────┘                                              │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │ 3. SORT       │  → 200 tasks sorted by created_at desc.     │
│  └──────┬────────┘                                              │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │ 4. PAGINATE   │  → 20 tasks returned.                        │
│  └──────┬────────┘                                              │
│         ▼                                                       │
│  Response: 20 of YOUR pending tasks, newest first. ✅           │
│                                                                 │
│  OWNERSHIP runs at step 1. Never step 2, 3, or 4.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ownership applies to single-resource endpoints too:**

```python
@router.get("/{task_id}", response_model=TaskResponse)
async def get_task(
    task_id: int,
    current_user: dict = Depends(get_current_user),
):
    task = next((t for t in ALL_TASKS if t["id"] == task_id), None)

    if task is None:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")

    # Ownership check — even for single-item fetch
    if current_user["role"] != "admin" and task["owner_id"] != current_user["id"]:
        # Return 404, not 403 — never confirm the resource EXISTS to unauthorised users
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")

    return task
```

> "Return `404`, not `403`, when an unauthorised user requests a resource they don't own. `403` tells the attacker 'this resource exists and you don't have access.' `404` tells them nothing. Information leakage is a real attack surface — don't hand it out voluntarily."

```
┌─────────────────────────────────────────────────────────────────┐
│                    ✏️  PRACTICE CHECKPOINT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q: A developer applies filters in this order:                  │
│                                                                 │
│     1. Filter by status (status=pending)                        │
│     2. Filter by priority (priority=3)                          │
│     3. Filter by owner_id (current_user["id"])  ← LAST         │
│                                                                 │
│     The result is correct. What's still wrong, and why          │
│     does the order matter even when the final data is the same? │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  SOLUTION                                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The final data IS the same. But the approach is wrong for      │
│  two reasons:                                                   │
│                                                                 │
│  1. PERFORMANCE: Steps 1 and 2 process ALL 50,000 tasks,        │
│     including 49,500 tasks that do not belong to this user.     │
│     You are doing expensive work on data you have no right      │
│     to return. With a real database, this means scanning and    │
│     loading rows you immediately discard.                       │
│                                                                 │
│  2. FUTURE BUGS: When you add sorting in step 3.5 and           │
│     pagination in step 4, inserting them BEFORE the ownership   │
│     filter is trivially easy. Then you've paginated across all  │
│     users' tasks and returned page 1 of the wrong superset.     │
│     The ownership filter being last is a footgun waiting         │
│     for a refactor.                                             │
│                                                                 │
│  Ownership MUST be step 1. It is the only safe order.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**For your Task Manager project:**

> "Replace the hardcoded `get_current_user` stub with your real JWT implementation in Week 9. The rest of the code changes nothing. The architecture is already correct — that is the point of the stub pattern."

---

---

# PART 5: IDEMPOTENCY

## 5.1 What Is Idempotency?

**The mathematical definition, made practical:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   IDEMPOTENCY                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  An operation is IDEMPOTENT if performing it ONCE has the       │
│  same effect as performing it MULTIPLE TIMES.                   │
│                                                                 │
│                                                                 │
│  REAL-WORLD EXAMPLES:                                           │
│  ─────────────────────                                          │
│  ✅ IDEMPOTENT:                                                  │
│  ├─ Setting thermostat to 72°F                                  │
│  │   (do it 5 times, temperature is still 72°F)                 │
│  ├─ Pressing elevator "up" button                               │
│  │   (pressing it 10 times doesn't call 10 elevators)           │
│  └─ Putting a letter in a specific mailbox                      │
│      (the letter is in that box regardless of how many          │
│       times you "put" it there)                                 │
│                                                                 │
│  ❌ NOT IDEMPOTENT:                                              │
│  ├─ Adding $100 to your bank account                            │
│  │   (do it 5 times = $500 added, not $100)                     │
│  ├─ Sending an email                                            │
│  │   (do it 3 times = recipient gets 3 emails)                  │
│  └─ Appending a line to a file                                  │
│      (do it twice = two identical lines)                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why does this matter for APIs?**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHY IDEMPOTENCY MATTERS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NETWORKS ARE UNRELIABLE.                                       │
│                                                                 │
│  Client ────── Request ──────▶ Server                           │
│         ◀───── Response ─────                                   │
│                                                                 │
│  What if the response gets lost?                                │
│                                                                 │
│  Client ────── Request ──────▶ Server  (processes it ✅)        │
│         ◀───── Respon ──✕  (network drops the response)         │
│                                                                 │
│  Client thinks: "Did it work? No response. I'll retry."        │
│                                                                 │
│  Client ────── SAME Request ─▶ Server  (processes it AGAIN?!)   │
│                                                                 │
│  If the operation is idempotent:                                │
│    Same result. No harm done. ✅                                 │
│                                                                 │
│  If the operation is NOT idempotent:                            │
│    User gets charged twice. Task created twice.                 │
│    Two emails sent. Data corrupted. 💥                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 HTTP Methods and Idempotency

**Connection to what you've learned:**

> "Remember the HTTP method semantics from Week 3 Lecture 1? We covered GET, POST, PUT, PATCH, DELETE. Now we add a new lens: which ones are SAFE to retry?"

```
┌─────────────────────────────────────────────────────────────────┐
│              HTTP METHODS & IDEMPOTENCY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  METHOD  │ IDEMPOTENT? │ SAFE?  │ WHY                          │
│  ────────┼─────────────┼────────┼─────────────────────────────  │
│  GET     │  ✅ Yes     │ ✅ Yes │ Reading doesn't change data   │
│          │             │        │ GET 10 times = same result    │
│          │             │        │                               │
│  PUT     │  ✅ Yes     │ ❌ No  │ "Set task status to DONE"     │
│          │             │        │ Do it 5 times = still DONE    │
│          │             │        │ Replaces the ENTIRE resource  │
│          │             │        │                               │
│  DELETE  │  ✅ Yes     │ ❌ No  │ "Delete task 42"              │
│          │             │        │ Do it 5 times = task 42 gone  │
│          │             │        │ (second+ calls: 404, but      │
│          │             │        │  the END STATE is the same)   │
│          │             │        │                               │
│  POST   │  ❌ No      │ ❌ No  │ "Create a new task"           │
│          │             │        │ Do it 5 times = 5 tasks! 💥   │
│          │             │        │                               │
│  PATCH   │  ⚠️ Depends │ ❌ No  │ "Increment priority by 1":   │
│          │             │        │  NOT idempotent (1→2→3→4...)  │
│          │             │        │ "Set priority to 3":          │
│          │             │        │  IS idempotent (3→3→3...)     │
│          │             │        │                               │
│  ────────┴─────────────┴────────┴─────────────────────────────  │
│                                                                 │
│  SAFE = Does not modify server state (read-only)                │
│  IDEMPOTENT = Multiple identical requests = same end state      │
│                                                                 │
│  Note: idempotent means same END STATE, not same RESPONSE.     │
│  DELETE /tasks/42:                                              │
│    First call  → 204 No Content (deleted)                       │
│    Second call → 404 Not Found (already gone)                   │
│    END STATE is the same: task 42 does not exist.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "PUT replaces the entire resource. If I `PUT /tasks/42` with `{title: 'New', status: 'done'}` five times... what's the end state?"

Answer: Task 42 has title "New" and status "done." Same as if you did it once. That's why PUT is idempotent.

> "What about PATCH? If I `PATCH /tasks/42` with `{priority: priority + 1}` five times?"

Answer: Priority goes from 1 → 2 → 3 → 4 → 5. Each call changes the state differently. That's NOT idempotent.

> "But if I `PATCH /tasks/42` with `{priority: 3}` five times?"

Answer: Priority is 3 every time. Same end state. That IS idempotent. This is why PATCH says "depends" — it depends on HOW you use it.

---

## 5.3 The Double-Click Disaster

**The most common real-world problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE DOUBLE-CLICK DISASTER                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User clicks "Create Task" button.                              │
│  Network is slow. Nothing seems to happen.                      │
│  User clicks again. And again.                                  │
│                                                                 │
│  Click 1 ────── POST /tasks {"title": "Deploy"} ──▶ Server     │
│  Click 2 ────── POST /tasks {"title": "Deploy"} ──▶ Server     │
│  Click 3 ────── POST /tasks {"title": "Deploy"} ──▶ Server     │
│                                                                 │
│  Result: THREE identical tasks created.                         │
│                                                                 │
│     ┌─────┬──────────────┬──────────┐                           │
│     │ ID  │ Title        │ Status   │                           │
│     ├─────┼──────────────┼──────────┤                           │
│     │ 101 │ Deploy       │ pending  │  ← Wanted this one       │
│     │ 102 │ Deploy       │ pending  │  ← Duplicate!            │
│     │ 103 │ Deploy       │ pending  │  ← Duplicate!            │
│     └─────┴──────────────┴──────────┘                           │
│                                                                 │
│  STORE ANALOGY:                                                 │
│  Clicking "Place Order" three times because the                 │
│  page seemed frozen. Three packages arrive.                     │
│  Three charges on your credit card.                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 Idempotency Keys (The Safety Net)

**The solution: give each intended action a unique ID.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   IDEMPOTENCY KEYS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HOW IT WORKS:                                                  │
│  ─────────────                                                  │
│  1. Client generates a unique key (UUID) BEFORE sending.        │
│  2. Client includes the key in a header.                        │
│  3. Server checks: "Have I seen this key before?"               │
│  4. If NO  → Process the request, store the key + result.       │
│  5. If YES → Return the stored result. Don't process again.     │
│                                                                 │
│                                                                 │
│  FLOW:                                                          │
│                                                                 │
│  Click 1:                                                       │
│  POST /tasks                                                    │
│  Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000          │
│  {"title": "Deploy"}                                            │
│  → Server: new key, process it → 201 Created (task #101)        │
│  → Server stores: key → {status: 201, body: {id: 101, ...}}    │
│                                                                 │
│  Click 2 (same key):                                            │
│  POST /tasks                                                    │
│  Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000          │
│  {"title": "Deploy"}                                            │
│  → Server: SEEN this key! Return stored result.                 │
│  → 201 Created (task #101) — same response, no new task.       │
│                                                                 │
│  Click 3 (same key):                                            │
│  → Same. No new task created. Always returns task #101.         │
│                                                                 │
│  RESULT: One task. No duplicates. Safe to retry. ✅              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Conceptual implementation:**

```python
from fastapi import APIRouter, Header, HTTPException
from uuid import UUID

router = APIRouter(prefix="/v1/tasks", tags=["tasks"])

# Simple in-memory store for idempotency keys
# (In production: Redis with TTL — you'll learn this in Week 10)
processed_keys: dict[str, dict] = {}


@router.post("", status_code=201)
async def create_task(
    task: TaskCreate,
    idempotency_key: str | None = Header(default=None),
):
    # If client provided an idempotency key, check for duplicates
    if idempotency_key is not None:
        if idempotency_key in processed_keys:
            # Already processed this request — return cached result
            return processed_keys[idempotency_key]

    # Process the request (create the task)
    new_task = {"id": generate_id(), "title": task.title, "status": "pending"}
    ALL_TASKS.append(new_task)

    # Cache the result if we have an idempotency key
    if idempotency_key is not None:
        processed_keys[idempotency_key] = new_task

    return new_task
```

**Important nuances:**

```
┌─────────────────────────────────────────────────────────────────┐
│              IDEMPOTENCY KEY RULES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Keys should EXPIRE after a reasonable time (e.g., 24 hrs)  │
│     Don't store them forever — that's a memory leak.            │
│     (You'll use Redis TTL for this in Week 10.)                │
│                                                                 │
│  2. Keys are PER-OPERATION, not per-endpoint.                   │
│     A new "create task" action = new key.                       │
│     A retry of the SAME action = same key.                      │
│                                                                 │
│  3. The CLIENT generates the key, not the server.               │
│     Usually a UUID. Generated before the first attempt.         │
│                                                                 │
│  4. Same key + DIFFERENT body = error (409 Conflict).           │
│     The key locks in the entire request. You can't reuse        │
│     a key with changed data.                                    │
│                                                                 │
│  5. Not every endpoint needs idempotency keys.                  │
│     GET, PUT, DELETE are already idempotent.                    │
│     Only POST (and sometimes PATCH) need them.                  │
│                                                                 │
│  USED BY: Stripe, PayPal, AWS, Shopify — any API               │
│  where duplicate operations would cause real damage.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**For your Task Manager project:**

> "You don't need to implement idempotency keys for this project — your in-memory Task Manager is low-stakes. But you MUST understand the concept. By Week 8 when you integrate external payment APIs, you'll see idempotency keys in the wild. And by Week 10 when you learn Redis, you'll have the tool to implement them properly."


## 5.5 Partial Updates with PATCH

**Connection to what you've learned:**

> "Part 5.2 taught you that PATCH is not always idempotent — it depends on HOW you use it. Part 5.3 warned about the double-click disaster for POST. Now: how do you actually implement a safe PATCH in FastAPI? The answer is in a Pydantic method you haven't used yet."

**The "wipe everything" bug:**

When every field in `TaskUpdate` is `Optional` with a default of `None`, a naive `model_dump()` will set every unspecified field to `None`, destroying existing data.

```python
# ❌ The Bug
class TaskUpdate(BaseModel):
    title: str | None = None
    status: str | None = None
    priority: int | None = None

# Client sends only the status change:
# PATCH /tasks/42
# {"status": "done"}

update = TaskUpdate(status="done")

print(update.model_dump())
# → {"title": None, "status": "done", "priority": None}
#               ────                  ────────────────
#               ↑ overwrites the existing title!  ↑ overwrites priority!
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE WIPE BUG                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Existing task in storage:                                      │
│  {"id": 42, "title": "Buy milk", "status": "pending",           │
│   "priority": 5}                                                │
│                                                                 │
│  Client sends: {"status": "done"}                               │
│                                                                 │
│  model_dump() produces:                                         │
│  {"title": None, "status": "done", "priority": None}            │
│                                                                 │
│  After task.update(update_data):                                │
│  {"id": 42, "title": None, "status": "done", "priority": None}  │
│              ────────────                     ─────────────     │
│              "Buy milk" is gone               5 is gone         │
│                                                                 │
│  The client changed ONE field and wiped TWO others. 💥          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The fix: `model_dump(exclude_unset=True)`**

```
┌─────────────────────────────────────────────────────────────────┐
│             model_dump() vs model_dump(exclude_unset=True)      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  model_dump()                                                   │
│  Dumps ALL fields, including those with their default values.   │
│  "title" was not sent → title is None (the default) → included  │
│                                                                 │
│  model_dump(exclude_unset=True)                                 │
│  Dumps ONLY fields that the client actually provided.           │
│  "title" was not sent → title is not in the dump at all         │
│                                                                 │
│                                                                 │
│  EXAMPLE:                                                       │
│                                                                 │
│  # Client sent: {"status": "done"}                              │
│  update = TaskUpdate(status="done")                             │
│                                                                 │
│  update.model_dump()                                            │
│  → {"title": None, "status": "done", "priority": None}         │
│     ──────────────                   ─────────────────          │
│     DANGER: will overwrite           DANGER: will overwrite     │
│                                                                 │
│  update.model_dump(exclude_unset=True)                          │
│  → {"status": "done"}                                           │
│     Only what was sent. Nothing else touched. ✅                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Full PATCH implementation:**

```python
from datetime import datetime
from fastapi import APIRouter, HTTPException

router = APIRouter(prefix="/v1/tasks", tags=["Tasks"])


@router.patch("/{task_id}", response_model=TaskResponse)
async def update_task(
    task_id: int,
    update: TaskUpdate,                 # All fields Optional — client sends only what changes
    current_user: dict = Depends(get_current_user),
):
    # Find the task
    task = next((t for t in ALL_TASKS if t["id"] == task_id), None)
    if task is None:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")

    # Ownership check (Part 4.6)
    if current_user["role"] != "admin" and task["owner_id"] != current_user["id"]:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")

    # Only update the fields the client actually sent
    update_data = update.model_dump(exclude_unset=True)

    if not update_data:
        # Client sent a PATCH with an empty body — that's a client error
        raise HTTPException(
            status_code=422,
            detail="PATCH body must contain at least one field to update.",
        )

    task.update(update_data)
    task["updated_at"] = datetime.utcnow().isoformat()

    return task


# ─── CONTRAST: PUT replaces the ENTIRE resource ───────────────────

@router.put("/{task_id}", response_model=TaskResponse)
async def replace_task(
    task_id: int,
    replacement: TaskCreate,            # Full resource required — no Optional fields
    current_user: dict = Depends(get_current_user),
):
    task = next((t for t in ALL_TASKS if t["id"] == task_id), None)
    if task is None:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")

    if current_user["role"] != "admin" and task["owner_id"] != current_user["id"]:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")

    # Replaces ALL mutable fields — omitting a field is not the same as keeping it
    task.update({
        "title": replacement.title,
        "description": replacement.description,
        "priority": replacement.priority,
        "updated_at": datetime.utcnow().isoformat(),
        # status and owner_id and created_at are preserved — they are immutable
    })
    return task
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  PUT vs PATCH — THE CLEAR RULE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PUT   → "Replace the resource with exactly what I send."       │
│          Input model: TaskCreate (all required fields)          │
│          Omitting a field means setting it to its default.      │
│          Client sends the FULL intended state.                  │
│          Idempotent ✅                                           │
│                                                                 │
│  PATCH → "Update only the fields I send."                       │
│          Input model: TaskUpdate (all Optional fields)          │
│          Omitting a field means leaving it unchanged.           │
│          Client sends ONLY the delta.                           │
│          Idempotent only if fields are set (not incremented) ⚠️  │
│                                                                 │
│  RULE: If your UI has an "Edit Task" form that submits the      │
│  full form every time, use PUT. If it sends only changed        │
│  fields (e.g., a live inline edit), use PATCH.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    ✏️  PRACTICE CHECKPOINT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q: A student writes this PATCH handler:                        │
│                                                                 │
│     @router.patch("/{task_id}")                                 │
│     async def update_task(task_id: int, update: TaskUpdate):    │
│         task = find_task(task_id)                               │
│         update_data = update.model_dump()    ← no exclude_unset │
│         for k, v in update_data.items():                        │
│             if v is not None:                ← they try to fix  │
│                 task[k] = v                                     │
│         return task                                             │
│                                                                 │
│     A client wants to set description=None (clear it).          │
│     They send: PATCH /tasks/42 {"description": null}            │
│                                                                 │
│     Does this work? What is the specific failure?               │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  SOLUTION                                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  It does NOT work. The `if v is not None` guard treats          │
│  an intentional null ("clear this field") the same as           │
│  an absent field ("don't touch this field").                    │
│                                                                 │
│  When the client sends {"description": null}:                   │
│  model_dump() → {"title": None, "description": None, ...}       │
│  The loop skips description because it's None.                  │
│  description in storage is unchanged. Silent failure.           │
│                                                                 │
│  The correct fix: model_dump(exclude_unset=True).               │
│  {"description": null} → {"description": None} → IS in the     │
│  dump because the client explicitly set it. The None guard      │
│  is then not needed. update task["description"] = None works.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**For your Task Manager project:**

> "Implement `PATCH /v1/tasks/{task_id}`. It should update only the provided fields, validate ownership (Part 4.6), and update `updated_at` on every successful write."

---

# PART 6: HATEOAS & API AS CONTRACT

## 6.1 HATEOAS: Teaching Your API to Give Directions

**HATEOAS = Hypermedia As The Engine Of Application State.**

Don't memorize the acronym. Understand the idea.

```
┌─────────────────────────────────────────────────────────────────┐
│                        HATEOAS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THE IDEA:                                                      │
│  ─────────                                                      │
│  An API response should include LINKS telling the client        │
│  what they can do NEXT. The client discovers available          │
│  actions from the response, not from reading documentation.     │
│                                                                 │
│                                                                 │
│  STORE ANALOGY:                                                 │
│  ──────────────                                                 │
│  WITHOUT HATEOAS:                                               │
│  You buy a product. The receipt says "Order #1234."             │
│  To track it? Figure out the URL yourself. Good luck.           │
│                                                                 │
│  WITH HATEOAS:                                                  │
│  You buy a product. The receipt says:                           │
│    Order #1234                                                  │
│    → Track this order: /orders/1234/tracking                    │
│    → Return this item: /orders/1234/returns                     │
│    → Buy again: /products/567                                   │
│  Every action is a click away.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Without HATEOAS — the client guesses:**

```json
{
    "id": 42,
    "title": "Write unit tests",
    "status": "pending",
    "category_id": 7
}
```

> Client thinks: "I have `category_id: 7`. How do I get the category details? Is it `/categories/7`? `/api/v1/categories/7`? `/category?id=7`? Time to read the docs."

**With HATEOAS — the API tells you:**

```json
{
    "id": 42,
    "title": "Write unit tests",
    "status": "pending",
    "category_id": 7,
    "links": {
        "self": "/v1/tasks/42",
        "category": "/v1/categories/7",
        "update": "/v1/tasks/42",
        "delete": "/v1/tasks/42",
        "tags": "/v1/tasks/42/tags"
    }
}
```

> Client thinks: "I see `category` link. I'll follow it." No guessing required.

**And in paginated responses:**

```json
{
    "items": [...],
    "pagination": {
        "next_cursor": "aWQ6MjA=",
        "has_more": true
    },
    "links": {
        "self": "/v1/tasks?limit=20",
        "next": "/v1/tasks?limit=20&cursor=aWQ6MjA=",
        "prev": null
    }
}
```

> "The `next` link means the client doesn't need to construct the pagination URL themselves. They just follow the link. If you change your cursor format, the client doesn't break because they never parse the cursor — they just follow the URL."

**The honest truth about HATEOAS:**

```
┌─────────────────────────────────────────────────────────────────┐
│                HATEOAS: THE REALITY CHECK                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IN THEORY: Beautiful. Fully self-discoverable APIs.            │
│  IN PRACTICE: Most APIs implement it partially, if at all.     │
│                                                                 │
│  FULL HATEOAS:                                                  │
│  ├─ GitHub API (link headers for pagination)                    │
│  ├─ PayPal API (links in every response)                        │
│  └─ Very few others do it completely                            │
│                                                                 │
│  PARTIAL HATEOAS (what most APIs do):                           │
│  ├─ Pagination links (next, prev) ← Very common                │
│  ├─ Self link (canonical URL of this resource) ← Common        │
│  └─ Related resource links ← Sometimes                         │
│                                                                 │
│  NO HATEOAS:                                                    │
│  └─ Most APIs rely on documentation instead of links            │
│                                                                 │
│                                                                 │
│  FOR YOUR PROJECT:                                              │
│  Implement pagination links (next, prev).                       │
│  Awareness of the full concept is enough for now.               │
│  Don't over-engineer it.                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 API Documentation Is a Contract

```
┌─────────────────────────────────────────────────────────────────┐
│              DOCUMENTATION AS CONTRACT                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your API documentation is a PROMISE to your consumers.         │
│                                                                 │
│  It says:                                                       │
│  ├─ "These endpoints exist."                                    │
│  ├─ "They accept these parameters."                             │
│  ├─ "They return these shapes."                                 │
│  ├─ "These are the possible errors."                            │
│  └─ "These are the rules (auth, rate limits, pagination)."      │
│                                                                 │
│  If your code does something different from your docs,          │
│  your code has a BUG — even if it "works."                      │
│                                                                 │
│                                                                 │
│  STORE ANALOGY:                                                 │
│  ──────────────                                                 │
│  The menu says "Pasta - $12." You order pasta.                  │
│  The bill says $18.                                             │
│  The restaurant says "the menu was outdated."                   │
│  That's not an explanation. That's a broken contract.           │
│                                                                 │
│  Same principle: if your docs say the endpoint returns          │
│  {status: string} and your code returns {state: {...}},         │
│  your API has a broken contract.                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.3 OpenAPI: The Contract You Already Have (Connection to FastAPI)

**Connection to what you've learned:**

> "Remember Week 3 Lecture 2 when you opened `/docs` in the browser and saw the Swagger UI? That's auto-generated from the **OpenAPI specification**. FastAPI builds this spec from your type hints, Pydantic models, and route definitions. You've been writing your API contract without knowing it."

```
┌─────────────────────────────────────────────────────────────────┐
│                 OPENAPI SPECIFICATION                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OpenAPI (formerly Swagger) is a STANDARD FORMAT for            │
│  describing REST APIs. It's a JSON/YAML file that says:         │
│                                                                 │
│  • What endpoints exist                                         │
│  • What parameters they accept                                  │
│  • What request bodies they expect                              │
│  • What responses they return (shapes + status codes)           │
│  • What authentication is required                              │
│  • What errors are possible                                     │
│                                                                 │
│                                                                 │
│  FASTAPI GIVES YOU THIS FOR FREE:                               │
│  ─────────────────────────────────                              │
│  /docs      → Swagger UI (interactive)                          │
│  /redoc     → ReDoc (clean documentation)                       │
│  /openapi.json → The raw spec (machine-readable)                │
│                                                                 │
│  Other teams can:                                               │
│  ├─ Read your docs to understand the API                        │
│  ├─ Import openapi.json into Postman for testing                │
│  ├─ Auto-generate client code from the spec                     │
│  └─ Run contract tests (does the API match the spec?)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Making your auto-docs USEFUL — not just generated:**

```python
from fastapi import APIRouter, Query
from pydantic import BaseModel, Field

router = APIRouter(
    prefix="/v1/tasks",
    tags=["Tasks"],  # Groups endpoints in Swagger UI
)


class TaskResponse(BaseModel):
    """A task in the system."""

    id: int = Field(..., description="Unique task identifier", examples=[42])
    title: str = Field(
        ..., min_length=1, max_length=200,
        description="Human-readable task title",
        examples=["Write unit tests for auth module"],
    )
    status: str = Field(
        ..., description="Current task status",
        examples=["pending"],
    )

    # Pydantic model_config can hold schema examples too
    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "id": 42,
                    "title": "Write unit tests",
                    "status": "pending",
                }
            ]
        }
    }


@router.get(
    "",
    response_model=PaginatedResponse[TaskResponse],
    summary="List tasks with pagination and filters",
    description=(
        "Retrieve a paginated list of tasks. Supports filtering by status, "
        "priority, and category. Results are sorted by the specified field."
    ),
    responses={
        200: {"description": "Paginated list of tasks"},
        422: {"description": "Invalid query parameters"},
    },
)
async def get_tasks(
    pagination: PaginationParams = Depends(),
    filters: TaskFilterParams = Depends(),
    sorting: SortParams = Depends(),
):
    ...
```

**Connection to Type Hints (Week 1 Lecture 1):**

> "Remember when we said type hints 'catch bugs before runtime'? Here's another payoff: every type hint you write becomes part of your OpenAPI spec. `int`, `str`, `Optional[str]`, `list[TaskResponse]` — all of it shows up in your docs automatically. Better type hints = better documentation = better API contract."

**What good auto-docs look like vs bad ones:**

```
┌─────────────────────────────────────────────────────────────────┐
│               BAD DOCS vs GOOD DOCS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BAD (default FastAPI with no effort):                          │
│  ─────────────────────────────────────                          │
│  GET /tasks                                                     │
│  Parameters: limit (integer), cursor (string)                   │
│  Response: 200                                                  │
│                                                                 │
│  (What does limit max at? What format is cursor?                │
│   What does the response look like? What errors can happen?)    │
│                                                                 │
│                                                                 │
│  GOOD (with descriptions, examples, constraints):               │
│  ─────────────────────────────────────────────────              │
│  GET /tasks — List tasks with pagination and filters            │
│                                                                 │
│  Retrieve a paginated list of tasks. Supports filtering         │
│  by status, priority, and category.                             │
│                                                                 │
│  Parameters:                                                    │
│  ├─ limit (integer, 1-100, default: 20)                         │
│  │   Number of items per page.                                  │
│  ├─ cursor (string, optional)                                   │
│  │   Opaque pagination token from previous response.            │
│  ├─ status (string, optional)                                   │
│  │   Filter by task status. Values: pending, done.              │
│  └─ sort_by (string, default: created_at)                       │
│      Sort field. Values: created_at, title, priority.           │
│                                                                 │
│  Response 200: { items: [...], pagination: {...} }              │
│  Response 422: Invalid parameter value                          │
│                                                                 │
│  Example response included.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "The difference between bad docs and good docs is 10 minutes of adding `description`, `examples`, and `responses` to your route definition. Your future self and every person who consumes your API will thank you."

---

# PART 7 [NEW]

## 7.1 The Missing Half of Your API

**The course so far taught you how to shape successful responses. It said nothing about failed ones.**

Look at Part 1.1's terrible API again:

```python
@app.post("/deleteTask")
async def delete_task(task_id: int):
    del TASKS[task_id]
    return {"msg": "ok"}      # ← "success" error message
```

And its HTTPException equivalent:

```python
raise HTTPException(status_code=404, detail="not found")
```

FastAPI's default 404 response looks like this:

```json
{
    "detail": "not found"
}
```

**Ask the class:**

> "You are building the frontend. The API returns `{"detail": "not found"}`. Which field wasn't found? Was it a task? A category? A user? Was the task ID wrong or did it get deleted? What should the user do next?"

Answer: **You have no idea.** The error tells you the HTTP status code and a string. That is all. Error responses in most APIs are a second-class citizen — verbose on success, useless on failure.

```
┌─────────────────────────────────────────────────────────────────┐
│               WHAT A GOOD ERROR RESPONSE NEEDS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Machine-readable error TYPE                                 │
│     So code can react differently to "not found" vs             │
│     "rate limited" vs "validation failed" without parsing       │
│     a human-readable string.                                    │
│                                                                 │
│  2. Human-readable TITLE                                        │
│     A stable, short label for this class of error.             │
│     "Task Not Found" — same for every 404 on this resource.    │
│                                                                 │
│  3. The HTTP STATUS CODE mirrored in the body                   │
│     Client libraries sometimes strip status codes.              │
│     The body should be self-contained.                          │
│                                                                 │
│  4. A DETAIL specific to this occurrence                        │
│     "Task with ID 42 does not exist." — not just "not found."   │
│                                                                 │
│  5. The INSTANCE URI                                            │
│     Which URL triggered this error? Invaluable in logs.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.2 RFC 7807: Problem Details Standard

**There is already a published standard for exactly this.** RFC 7807 — "Problem Details for HTTP APIs" — defines a `application/problem+json` response format that every major API provider either follows or deviates from intentionally.

```
┌─────────────────────────────────────────────────────────────────┐
│                  RFC 7807 PROBLEM DETAILS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  {                                                              │
│    "type":     "https://api.example.com/problems/not-found",    │
│                 ──────────────────────────────────────────       │
│                 URI that identifies the PROBLEM TYPE.           │
│                 Clients use this to switch on error type,        │
│                 not on the status code or detail string.         │
│                 Should resolve to documentation if visited.      │
│                                                                 │
│    "title":    "Task Not Found",                                │
│                 ──────────────                                   │
│                 Stable, human-readable summary.                 │
│                 SAME for every 404 on tasks. Do not include     │
│                 dynamic values here — that's what detail is for. │
│                                                                 │
│    "status":   404,                                             │
│                 ───                                             │
│                 The HTTP status code. Must match the            │
│                 response's actual status code.                  │
│                                                                 │
│    "detail":   "Task with ID 42 does not exist.",               │
│                 ──────────────────────────────────               │
│                 Human-readable. Specific to this request.        │
│                 May include dynamic values (the ID, the input).  │
│                                                                 │
│    "instance": "/v1/tasks/42"                                   │
│                 ─────────────                                   │
│                 The URI that caused the error.                  │
│                 Optional but invaluable for support tickets.    │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Validation errors extend the base format:**

```json
{
    "type": "https://api.example.com/problems/validation-error",
    "title": "Request Validation Failed",
    "status": 422,
    "detail": "2 validation error(s) found.",
    "instance": "/v1/tasks",
    "errors": [
        {
            "field": "title",
            "message": "Field required",
            "loc": ["body", "title"]
        },
        {
            "field": "priority",
            "message": "Input should be less than or equal to 10",
            "loc": ["body", "priority"]
        }
    ]
}
```

> "Notice the `errors` array — every validation error is returned at once, not just the first. A frontend form can highlight all invalid fields simultaneously, not force the user through one error at a time. This pays off the `RequestValidationError` tech debt from Week 3."

---

## 7.3 Implementing Error Envelopes in FastAPI

**Connection to what you've learned:**

> "Remember `@app.exception_handler` from Week 3 Lecture 4? You used it to override FastAPI's default error format. That is precisely where error envelopes belong."

```python
# src/api/errors.py
from pydantic import BaseModel


class ProblemDetail(BaseModel):
    """RFC 7807 Problem Details — base format for all API errors."""
    type: str
    title: str
    status: int
    detail: str
    instance: str | None = None

    model_config = {
        "json_schema_extra": {
            "examples": [{
                "type": "https://api.example.com/problems/not-found",
                "title": "Task Not Found",
                "status": 404,
                "detail": "Task with ID 42 does not exist.",
                "instance": "/v1/tasks/42",
            }]
        }
    }


class ValidationProblemDetail(ProblemDetail):
    """Extended Problem Details for request validation failures."""
    errors: list[dict]


# ─── Helper ──────────────────────────────────────────────────────

_STATUS_TITLES: dict[int, str] = {
    400: "Bad Request",
    401: "Not Authenticated",
    403: "Forbidden",
    404: "Not Found",
    409: "Conflict",
    410: "Gone",
    422: "Request Validation Failed",
    429: "Too Many Requests",
    500: "Internal Server Error",
    503: "Service Unavailable",
}

def _status_title(code: int) -> str:
    return _STATUS_TITLES.get(code, f"HTTP Error {code}")
```

```python
# src/main.py
from fastapi import FastAPI, HTTPException, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

from src.api.errors import ProblemDetail, ValidationProblemDetail, _status_title

app = FastAPI()

PROBLEM_CONTENT_TYPE = "application/problem+json"


@app.exception_handler(HTTPException)
async def http_exception_handler(
    request: Request,
    exc: HTTPException,
) -> JSONResponse:
    """Converts ALL HTTPExceptions into RFC 7807 Problem Details."""
    problem = ProblemDetail(
        type=f"https://api.example.com/problems/{exc.status_code}",
        title=_status_title(exc.status_code),
        status=exc.status_code,
        detail=str(exc.detail),
        instance=str(request.url),
    )
    return JSONResponse(
        status_code=exc.status_code,
        content=problem.model_dump(),
        headers={"Content-Type": PROBLEM_CONTENT_TYPE},
    )


@app.exception_handler(RequestValidationError)
async def validation_exception_handler(
    request: Request,
    exc: RequestValidationError,
) -> JSONResponse:
    """
    Collects ALL validation errors (not just the first) and returns them
    in a structured Problem Details response.
    Pays off: RequestValidationError full error list (W3L4 tech debt).
    """
    errors = []
    for error in exc.errors():   # ← iterate ALL errors, not just errors()[0]
        # Normalise the location path to remove internal pydantic prefixes
        loc = list(error.get("loc", []))
        # loc[0] is usually "body", "query", "path" — keep it for context
        field = str(loc[-1]) if loc else "unknown"

        errors.append({
            "field": field,
            "message": error["msg"],
            "loc": loc,              # Full path: ["body", "priority"]
        })

    problem = ValidationProblemDetail(
        type="https://api.example.com/problems/validation-error",
        title="Request Validation Failed",
        status=422,
        detail=f"{len(errors)} validation error(s). Check the 'errors' field.",
        instance=str(request.url),
        errors=errors,
    )
    return JSONResponse(
        status_code=422,
        content=problem.model_dump(),
        headers={"Content-Type": PROBLEM_CONTENT_TYPE},
    )
```

**Connecting the error envelope to your router's `responses` parameter (from Part 2.6):**

```python
# Now that ProblemDetail is defined, use it in include_router() responses
# so OpenAPI documents it properly for every route in your router:

from src.api.errors import ProblemDetail

app.include_router(
    tasks_router,
    prefix="/v1/tasks",
    tags=["Tasks"],
    responses={
        401: {"model": ProblemDetail, "description": "Not authenticated."},
        403: {"model": ProblemDetail, "description": "Insufficient permissions."},
        404: {"model": ProblemDetail, "description": "Task not found."},
        422: {"model": ValidationProblemDetail, "description": "Validation failed."},
    },
)
```

> "Your OpenAPI spec now documents both successful and failed response shapes for every endpoint. Consumers know exactly what to expect regardless of outcome. The API contract is complete."

**What happens to a 500 (unhandled exception)?**

```python
@app.exception_handler(Exception)
async def unhandled_exception_handler(
    request: Request,
    exc: Exception,
) -> JSONResponse:
    """
    Catch-all for unhandled exceptions.
    In production: log the real error internally. Return a generic 500.
    NEVER expose internal error details (stack traces) to the client.
    """
    # structlog.error("Unhandled exception", exc_info=exc)  ← Week 15 topic
    problem = ProblemDetail(
        type="https://api.example.com/problems/internal-error",
        title="Internal Server Error",
        status=500,
        detail="An unexpected error occurred. Our team has been notified.",
        instance=str(request.url),
    )
    return JSONResponse(
        status_code=500,
        content=problem.model_dump(),
        headers={"Content-Type": PROBLEM_CONTENT_TYPE},
    )
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    ✏️  PRACTICE CHECKPOINT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q: A developer writes this in a route handler:                 │
│                                                                 │
│     raise HTTPException(                                        │
│         status_code=404,                                        │
│         detail="task not found"                                 │
│     )                                                           │
│                                                                 │
│     After adding the error envelope handler from Part 7.3,      │
│     what does the response body look like? Fill in the values.  │
│                                                                 │
│     {                                                           │
│         "type": "___",                                          │
│         "title": "___",                                         │
│         "status": ___,                                          │
│         "detail": "___",                                        │
│         "instance": "___"                                       │
│     }                                                           │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  SOLUTION                                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  {                                                              │
│      "type":     "https://api.example.com/problems/404",        │
│      "title":    "Not Found",                                   │
│      "status":   404,          ← mirrors the HTTP status code   │
│      "detail":   "task not found",   ← exc.detail verbatim     │
│      "instance": "/v1/tasks/42"      ← the request URL          │
│  }                                                              │
│                                                                 │
│  Content-Type header: application/problem+json                  │
│  HTTP status code:    404                                       │
│                                                                 │
│  Note: "title" comes from _status_title(404) → "Not Found",    │
│  NOT from exc.detail. detail is where the specific message goes. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**For your Task Manager project:**

> "Add the two exception handlers to `main.py`. Verify that hitting a nonexistent task ID returns a `ProblemDetail` JSON body with `Content-Type: application/problem+json`. Update your test suite: every test that checks error responses should assert the body matches `ProblemDetail` shape, not just the status code."

---

# PART 8 [NEW]

## 8.1 The "Why Fetch Again?" Problem

**Connection to the "Efficient" quality from Part 1.2:**

Part 3 solved the problem of over-fetching data (50,000 items vs 20). There is a second efficiency problem: fetching data that **hasn't changed**.

```
┌─────────────────────────────────────────────────────────────────┐
│               THE UNNECESSARY RE-FETCH PROBLEM                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User opens their Task Manager.                                 │
│  Browser fetches GET /tasks/42.    → 200 OK, 1.2 KB             │
│                                                                 │
│  User glances away for 3 seconds.                               │
│  User looks back. Browser re-fetches GET /tasks/42. → 200 OK   │
│                                                                 │
│  Has the task changed? No.                                      │
│  Did you send 1.2 KB anyway? Yes.                               │
│  Did the server spend time serialising the same data? Yes.      │
│                                                                 │
│  On desktop with fast wifi: invisible.                          │
│  On mobile with 4G: costs bandwidth.                            │
│  On a SPA that polls every 5 seconds: 1.2 KB × 12 × 60 per hr  │
│  = 864 KB/hr of unchanged data per user.                        │
│                                                                 │
│  STORE ANALOGY:                                                 │
│  Refreshing a product page reloads the entire page from         │
│  the server even if the price hasn't changed in months.         │
│  A well-designed store serves a cached version until            │
│  something actually changes.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The solution: let the client say "I already have a version — is it still current?"**

---

## 8.2 ETag: The Fingerprint

```
┌─────────────────────────────────────────────────────────────────┐
│                       ETAG FLOW                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: First request (client has nothing cached)              │
│                                                                 │
│  Client ── GET /tasks/42 ──────────────────────────────▶ Server │
│  Server ◀─ 200 OK + body + ETag: "a3f8b21c" ────────────────── │
│                                                                 │
│  Client stores: (url → body, etag → "a3f8b21c")                │
│                                                                 │
│                                                                 │
│  STEP 2: Second request (client has a cached version)           │
│                                                                 │
│  Client ── GET /tasks/42 ──────────────────────────────▶ Server │
│            If-None-Match: "a3f8b21c"                            │
│                                                                 │
│  [Task has NOT changed since Step 1]                            │
│  Server ◀─ 304 Not Modified (empty body) ───────────────────── │
│  Client: use cached version. Nothing transferred. ✅            │
│                                                                 │
│  [Task HAS changed since Step 1]                                │
│  Server ◀─ 200 OK + NEW body + ETag: "9d2e4a77" ─────────────  │
│  Client: update cache. ✅                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What is the ETag?**

An ETag (Entity Tag) is a fingerprint of the response body — typically a short hash. If the resource content changes, the hash changes. If content is identical, the hash is identical.

```
┌─────────────────────────────────────────────────────────────────┐
│                   ETAG PROPERTIES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPAQUE: The client treats it as a magic token.                  │
│          It does not parse it, interpret it, or construct it.   │
│          It only stores it and sends it back verbatim.           │
│                                                                 │
│  QUOTED: Always surrounded by double quotes in the header.      │
│          ETag: "a3f8b21c"                                       │
│          NOT: ETag: a3f8b21c                                    │
│                                                                 │
│  STRONG vs WEAK:                                                │
│  ├─ Strong (default): "a3f8b21c"                                │
│  │   Byte-for-byte identical. Safe for range requests.          │
│  └─ Weak: W/"a3f8b21c"                                          │
│      Semantically equivalent but not byte-identical.            │
│      (e.g., gzip vs plaintext of the same content)              │
│                                                                 │
│  You will almost always use strong ETags. Use the W/ prefix     │
│  only if you know you need it.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Cache-Control headers you need to know:**

```
┌─────────────────────────────────────────────────────────────────┐
│               CACHE-CONTROL DIRECTIVES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  private           Only the end client may cache.               │
│                    CDNs and shared proxies must not.            │
│                    Use for user-specific data (tasks, profiles). │
│                                                                 │
│  public            CDNs and shared proxies may also cache.      │
│                    Use for shared, non-user-specific data.      │
│                                                                 │
│  no-store          Never cache. Not even in the browser.        │
│                    Use for sensitive data (auth tokens, PII).   │
│                                                                 │
│  no-cache          Can cache, but MUST validate before using.   │
│                    (Despite the name, it does not prevent        │
│                     caching — it prevents using a stale cache.) │
│                                                                 │
│  max-age=N         Cache is fresh for N seconds. Do not ask     │
│                    the server during this window.               │
│                                                                 │
│  must-revalidate   Once max-age expires, MUST check with server │
│                    before using. Do not serve stale on error.   │
│                                                                 │
│  FOR YOUR TASK MANAGER:                                         │
│  Cache-Control: private, max-age=0, must-revalidate             │
│  ├─ private: tasks are user-specific, never cache at CDN        │
│  ├─ max-age=0: do not serve cached version automatically        │
│  └─ must-revalidate: always check ETag before using cache       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.3 Implementing ETags in FastAPI

```python
# src/api/cache.py
import hashlib
import json


def generate_etag(data: dict | list) -> str:
    """
    Generate a strong ETag from response data.
    The JSON is sorted by key so identical data always produces the same hash
    regardless of dictionary insertion order.
    """
    serialised = json.dumps(data, sort_keys=True, default=str)
    digest = hashlib.sha256(serialised.encode()).hexdigest()[:16]
    return f'"{digest}"'          # ETags MUST be quoted


def check_etag(request: Request, etag: str) -> bool:
    """
    Returns True if the client's cached version matches the current ETag.
    If True, the endpoint should return 304 Not Modified.
    """
    client_etag = request.headers.get("If-None-Match")
    return client_etag == etag
```

**Applying ETags to a single-resource endpoint:**

```python
from fastapi import APIRouter, HTTPException, Request
from fastapi.responses import JSONResponse, Response

from src.api.cache import generate_etag, check_etag

router = APIRouter(prefix="/v1/tasks", tags=["Tasks"])


@router.get("/{task_id}", response_model=TaskResponse)
async def get_task(
    task_id: int,
    request: Request,              # ← Need request to read If-None-Match header
    current_user: dict = Depends(get_current_user),
):
    task = next((t for t in ALL_TASKS if t["id"] == task_id), None)

    if task is None:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")

    if current_user["role"] != "admin" and task["owner_id"] != current_user["id"]:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")

    etag = generate_etag(task)

    # Client already has this exact version — save the bandwidth
    if check_etag(request, etag):
        return Response(status_code=304)      # No body. No Content-Type.

    return JSONResponse(
        content=task,
        headers={
            "ETag": etag,
            "Cache-Control": "private, max-age=0, must-revalidate",
        },
    )
```

**The full request/response sequence in practice:**

```python
# First request:
# GET /v1/tasks/42
# (no If-None-Match header)
#
# Response:
# HTTP/1.1 200 OK
# ETag: "a3f8b21cd4e5f678"
# Cache-Control: private, max-age=0, must-revalidate
# Content-Type: application/json
#
# {"id": 42, "title": "Write unit tests", "status": "pending", ...}


# Second request (task unchanged):
# GET /v1/tasks/42
# If-None-Match: "a3f8b21cd4e5f678"
#
# Response:
# HTTP/1.1 304 Not Modified
# ETag: "a3f8b21cd4e5f678"
# (no body)


# After a PATCH /v1/tasks/42 {"status": "done"}:
# GET /v1/tasks/42
# If-None-Match: "a3f8b21cd4e5f678"     ← stale etag
#
# Response:
# HTTP/1.1 200 OK
# ETag: "9d2e4a77b3c1e820"              ← new etag (task changed)
# Cache-Control: private, max-age=0, must-revalidate
#
# {"id": 42, "title": "Write unit tests", "status": "done", ...}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    ✏️  PRACTICE CHECKPOINT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q: A route handler does this:                                  │
│                                                                 │
│     etag = generate_etag(task)                                  │
│     if check_etag(request, etag):                               │
│         return Response(status_code=304)                        │
│     return task      ← returns the Pydantic model directly      │
│                                                                 │
│     The handler returns the task directly (not JSONResponse).   │
│     The ETag header is missing from the 200 response.           │
│     What two things are wrong, and what is the consequence?     │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  SOLUTION                                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WRONG 1: The 200 OK response has no ETag header.               │
│  Consequence: The client receives the data but has nothing      │
│  to store as a cache key. On the NEXT request, it cannot send   │
│  If-None-Match. Every subsequent request gets a full 200 OK.   │
│  ETags only work if the server ALWAYS sends them on 200.        │
│                                                                 │
│  WRONG 2: The 200 response has no Cache-Control header.         │
│  Consequence: The browser's default caching policy applies,     │
│  which varies by browser and can lead to aggressive caching     │
│  (serving stale task data) or no caching at all.               │
│                                                                 │
│  Fix: return JSONResponse(content=task.model_dump(),            │
│           headers={"ETag": etag,                                │
│                    "Cache-Control": "private, max-age=0,        │
│                                      must-revalidate"})          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When NOT to use ETags on your endpoints:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ETAG APPLICABILITY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ Single-resource GET:   GET /tasks/42                        │
│     The data is stable and self-contained. ETag is trivial.    │
│                                                                 │
│  ⚠️  Collection GET:       GET /tasks?status=pending           │
│     The ETag must hash ALL filtered, sorted results.            │
│     Adding one task invalidates the ETag for the whole page.   │
│     Use with caution — the cache hit rate will be low if        │
│     data changes frequently.                                    │
│                                                                 │
│  ❌ POST / PATCH / DELETE:                                       │
│     Write operations should not send ETags on the response      │
│     (with one exception: If-Match for optimistic concurrency   │
│      — covered in Week 7, Advanced DB Patterns).               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**For your Task Manager project:**

> "Add ETags to `GET /v1/tasks/{task_id}`. Write a test that: (1) fetches a task and captures the ETag; (2) sends the ETag in `If-None-Match` and asserts a 304 status with an empty body; (3) patches the task and then repeats the request with the OLD ETag and asserts a 200 with the new ETag."

---

# QUICK REFERENCE CARD — FULL REPLACEMENT

```
┌─────────────────────────────────────────────────────────────────┐
│                API DESIGN QUICK REFERENCE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  VERSIONING:                                                    │
│      /api/v1/tasks                  ← URL path (recommended)   │
│      Accept: application/vnd.v2     ← Header (know it exists)  │
│      /api/tasks?version=2           ← Query (avoid usually)    │
│                                                                 │
│  PAGINATION (Cursor):                                           │
│      GET /tasks?limit=20                     ← First page      │
│      GET /tasks?limit=20&cursor=abc123       ← Next page       │
│      Response: { items, pagination: { next_cursor, has_more } } │
│                                                                 │
│  PAGINATION (Offset):                                           │
│      GET /tasks?limit=20&offset=0            ← Page 1          │
│      GET /tasks?limit=20&offset=20           ← Page 2          │
│      Response: { items, total, limit, offset }                  │
│                                                                 │
│  FILTERING:                                                     │
│      GET /tasks?status=pending               ← Exact match     │
│      GET /tasks?search=deploy                ← Text search     │
│      GET /tasks?created_after=2025-01-01     ← Date range      │
│                                                                 │
│  SORTING:                                                       │
│      GET /tasks?sort_by=created_at&order=desc                   │
│                                                                 │
│  ORDER OF OPERATIONS (filter → sort → paginate):               │
│      OWNERSHIP → FILTER → SORT → PAGINATE (always this order)  │
│                                                                 │
│  NESTED RESOURCES vs QUERY PARAMS:                              │
│      Nested:  GET /tasks/{id}/comments  (child needs parent)    │
│      Flat:    GET /comments?task_id=42  (child is independent)  │
│      Rule:    Never more than 2 levels of nesting.              │
│                                                                 │
│  INPUT/OUTPUT MODEL SEPARATION:                                 │
│      TaskCreate   → POST body (only settable fields)            │
│      TaskUpdate   → PATCH body (all Optional, exclude_unset)    │
│      TaskResponse → all GET responses (full resource view)      │
│      Never use the same model for input and output.             │
│                                                                 │
│  IDEMPOTENCY:                                                   │
│      GET, PUT, DELETE → Already idempotent                      │
│      POST → Use Idempotency-Key header for safety               │
│      PATCH → Idempotent if setting, not if incrementing         │
│                                                                 │
│  PATCH (partial update):                                        │
│      update.model_dump(exclude_unset=True)  ← THE critical call │
│      Never: model_dump() + if v is not None guard               │
│                                                                 │
│  HATEOAS:                                                       │
│      Include pagination links (next, prev) at minimum           │
│                                                                 │
│  ERROR ENVELOPES (RFC 7807):                                    │
│      Content-Type: application/problem+json                     │
│      Body: { type, title, status, detail, instance }            │
│      Validation errors: extend with "errors": [...]             │
│      status in body MUST match HTTP status code                 │
│                                                                 │
│  HTTP CACHING (ETags):                                          │
│      200 response: ETag: "hash" + Cache-Control header          │
│      Client re-request: If-None-Match: "hash"                   │
│      Unchanged: 304 Not Modified (empty body)                   │
│      Changed:   200 OK + new body + new ETag                    │
│      User data: Cache-Control: private, max-age=0, must-revalidate│
│                                                                 │
│  OWNERSHIP:                                                     │
│      Filter by owner first, before any other filter             │
│      Single-resource 404 (not 403) for unauthorised access      │
│      Never expose that a resource EXISTS to unauthorised users  │
│                                                                 │
│  DOCUMENTATION:                                                 │
│      Add description, examples, and response codes              │
│      to every endpoint. /docs is your contract.                 │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ Returning all items with no limit  → Always paginate    │
│      ❌ No default sort order              → Results are random │
│      ❌ Filtering after pagination         → Wrong count/items  │
│      ❌ No max on limit parameter          → limit=999999       │
│      ❌ Breaking changes without new version → Clients crash    │
│      ❌ Same model for input and output    → Mass assignment    │
│      ❌ model_dump() in PATCH handler      → Wipes unset fields │
│      ❌ 403 for resource not owned         → Reveals existence  │
│      ❌ ETag on 200 but not set header     → Cache never works  │
│      ❌ Ownership filter after other filters → CPU waste + bug  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# SUMMARY — FULL REPLACEMENT

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  API DESIGN = EMPATHY FOR YOUR CONSUMER                         │
│                                                                 │
│  Every principle covered today is about making life             │
│  easier for the person CALLING your API:                        │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  Versioning  │  │  Pagination  │  │  Filtering   │           │
│  │              │  │              │  │  & Sorting   │           │
│  │  "I won't   │  │  "I'll give  │  │  "I'll find  │           │
│  │   break     │  │   you just   │  │   what you   │           │
│  │   your app" │  │   enough"    │  │   actually   │           │
│  │             │  │              │  │   need"      │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
│         │                 │                 │                    │
│         └─────────────────┼─────────────────┘                    │
│                           │                                     │
│                    ┌──────┴───────┐                              │
│                    │  YOUR  API   │                              │
│                    └──────┬───────┘                              │
│                           │                                     │
│         ┌─────────────────┼─────────────────┐                    │
│         │                 │                 │                    │
│  ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐           │
│  │ Idempotency  │  │   HATEOAS    │  │Documentation │           │
│  │  + PATCH     │  │              │  │              │           │
│  │  "I'm safe  │  │  "I'll tell  │  │  "I'll      │           │
│  │   to retry" │  │   you what's │  │   explain   │           │
│  │   to update"│  │   next"      │  │   myself"   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                 │
│         ┌─────────────────┬─────────────────┐                    │
│         │                 │                 │                    │
│  ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐           │
│  │    Error     │  │ HTTP Caching │  │  Ownership   │           │
│  │  Envelopes   │  │    ETags     │  │  + I/O Sep.  │           │
│  │  "I'm clear │  │  "I don't   │  │  "I know     │           │
│  │   when I    │  │   waste your │  │   whose data │           │
│  │   fail"     │  │   bandwidth" │  │   is whose"  │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                 │
│                                                                 │
│  THE ONLINE STORE ANALOGY — COMPLETE:                           │
│  ├─ Versioning   = Old bookmarks still work after redesign      │
│  ├─ Pagination   = 20 products per page, not 50,000             │
│  ├─ Filtering    = Sidebar filters narrow results                │
│  ├─ Sorting      = Sort dropdown organises results               │
│  ├─ Nesting      = /orders/42/items (items live inside orders)   │
│  ├─ Ownership    = Only you can see your own order history       │
│  ├─ I/O Sep.     = Store sets price; you pick quantity only      │
│  ├─ PATCH        = Change just your delivery address, not order  │
│  ├─ Idempotency  = "Place Order" twice only charges once         │
│  ├─ Error Envel. = Clear error page with reason and next steps   │
│  ├─ HTTP Caching = Browser reuses cached page if product unchanged│
│  ├─ HATEOAS      = "You might also like..." links                │
│  └─ Docs         = The store directory and help centre           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Upcoming Lectures and Project

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THIS WEEK (Project):                                           │
│  └─ Complete your Task Manager API with:                        │
│     ├─ API versioning (/v1/ prefix via APIRouter)               │
│     ├─ Cursor-based pagination on all list endpoints            │
│     ├─ Filtering and sorting on tasks                           │
│     └─ Comprehensive test suite covering all of the above       │
│                                                                 │
│  WEEK 5-6 (Databases):                                          │
│  └─ Pagination becomes a DATABASE concern                       │
│     OFFSET → SQL OFFSET (and why it's slow)                     │
│     CURSOR → WHERE id > last_seen_id (and why it's fast)        │
│     Filtering → SQL WHERE clauses                               │
│     Sorting → SQL ORDER BY                                      │
│                                                                 │
│  WEEK 8 (External APIs):                                        │
│  └─ You'll CONSUME APIs that use these patterns                 │
│     Handling pagination from third-party APIs                   │
│     Respecting rate limits (idempotency on your side)           │
│                                                                 │
│  WEEK 10 (Redis):                                               │
│  └─ Idempotency keys stored in Redis with TTL                   │
│     Rate limiting implemented with Redis counters               │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ Every pattern from today applied at scale                   │
│     Multi-tenant API versioning                                 │
│     Paginated search across entities                            │
│     Full OpenAPI documentation as deliverable                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```