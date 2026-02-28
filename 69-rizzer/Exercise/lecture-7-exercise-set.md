# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "HTTP is a complex binary protocol that libraries and browsers handle invisibly. PUT and PATCH both 'update' data, so they're interchangeable. POST is safe to call more than once if the data is the same. CORS is a security wall on the server that blocks unauthorized external attackers. Status codes are arbitrary numbers I look up in a reference table when something breaks."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---|---|
| 1 | HTTP is binary/magic — libraries handle mysterious low-level communication | HTTP/1.1 is plain, human-readable text. You can write a valid HTTP request by hand with a socket and a text string |
| 2 | PUT and PATCH both "update" a resource — they differ only in style or preference | PUT *replaces the entire resource*; unmentioned fields are permanently deleted. PATCH modifies only the specified fields |
| 3 | Sending the same POST request twice is harmless if the data is identical | POST is NOT idempotent — every call creates a new resource regardless of an identical payload |
| 4 | 401 and 403 both mean "access denied" | 401 = "I don't know WHO you are" (unauthenticated). 403 = "I know who you are, but NO" (unauthorised) |
| 5 | CORS is a server-side security wall that blocks external attackers from calling your API | CORS is enforced exclusively by browsers. curl, Postman, and Python scripts bypass it entirely. It protects the browser user, not the server |
| 6 | `Cache-Control: no-cache` means "never cache this response" | `no-cache` = "store it, but revalidate before using." `no-store` = "never cache at all." The names are deliberately misleading |
| 7 | Statelessness means the server has no memory of users at all | Statelessness means the server holds no between-request state *in process memory*. User state lives in the database or inside tokens the client sends with every request |
| 8 | A 5xx error means the client sent bad data | 4xx = the client made a mistake. 5xx = the server has a bug. A 500 is always the server team's problem to fix |

**The Prediction Gap:**
When a developer sends `PUT /books/42` with only `{"author": "New Author"}`, they expect only the author field to update. The actual result is that every other field — `title`, `year`, `available` — is permanently erased, because PUT means "replace the entire resource with exactly this." This is one of the most destructive bugs in production API usage and it runs silently with a 200 OK status.

---

## The Main Purpose (Essence)

**Core Insight:** HTTP is a human-designed, text-based conversation protocol with precise semantics. Every method, status code, and header exists to communicate *intent* and *outcome* unambiguously. Treating them as interchangeable or decorative destroys debuggability, breaks client contracts, and creates production disasters.

**The Problems Being Solved:**

| Concept | Without It | With It |
|---|---|---|
| HTTP Methods | Server must guess if you're creating, replacing, or deleting | The METHOD signals intent; the server knows exactly what to do |
| Status Codes | Clients parse body text to detect errors | Clients check the status code number — universal, language-agnostic |
| Idempotency | Retry logic on network failure creates duplicate orders, charges, emails | Safe retries for idempotent methods; idempotency keys make POST safe |
| CORS | Browser JavaScript can silently call any API using a victim's cookies | Browser enforces cross-origin consent before exposing responses to scripts |

**The Key Distinction:**
- **Method** = WHAT you want to do (GET, POST, PUT, PATCH, DELETE)
- **URL** = TO WHICH resource you want to do it
- **Status code** = WHAT HAPPENED as a result
- **Headers** = HOW to interpret the request and response
- **Body** = THE CONTENT being transferred

All five must be semantically correct simultaneously for an endpoint to be well-designed.

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Vanishing Book

```python
import httpx
import asyncio

# Current state of book #42 in the database:
# {
#   "id": 42,
#   "title": "Clean Code",
#   "author": "Robert Martin",
#   "year": 2008,
#   "available": True
# }

async def fix_author_name():
    async with httpx.AsyncClient() as client:

        # Developer wants to fix a minor name inconsistency
        response_put = await client.put(
            "http://api.library.com/books/42",
            json={"author": "Robert C. Martin"}  # Only sending the changed field
        )
        print(f"PUT status:  {response_put.status_code}")
        print(f"PUT result:  {response_put.json()}")

        # Separately, a borrower returns the book
        response_patch = await client.patch(
            "http://api.library.com/books/42",
            json={"available": True}
        )
        print(f"PATCH status: {response_patch.status_code}")
        print(f"PATCH result: {response_patch.json()}")

asyncio.run(fix_author_name())
```

**Questions:**

1. **Predict the PUT result.** What does the complete book record look like in the database immediately after the PUT request? What happened to `title`, `year`, and `available`?

2. **Predict the PATCH result.** After the subsequent PATCH, what does the record look like? (Note: what record is the PATCH being applied to?)

3. **Explain why.** A junior developer insists: "I only sent `author` in the PUT body, so only `author` should change — that's the same logic as PATCH." Explain precisely why this reasoning is wrong. What does PUT actually mean at the protocol level?

4. **Production consequence.** If this PUT ran against a real production database, describe the data loss event that occurred and what recovery would require.

---

## Exercise 1.2: The Retry Loop That Charged Twice

```python
import httpx
import asyncio

async def place_order(customer_id: str, item_id: str, amount_cents: int) -> dict:
    """Place a store order. Retries once on failure."""
    async with httpx.AsyncClient() as client:
        for attempt in range(2):
            try:
                response = await client.post(
                    "http://api.store.com/orders",
                    json={
                        "customer_id": customer_id,
                        "item_id": item_id,
                        "amount_cents": amount_cents
                    },
                    timeout=5.0
                )
                if response.status_code == 201:
                    return response.json()

            except httpx.TimeoutException:
                print(f"Attempt {attempt + 1} timed out. Retrying...")
                continue

    return {"error": "Order failed after retries"}

# Network scenario:
# The FIRST request reaches the server and SUCCEEDS.
# The server creates Order #99 and charges the customer's card.
# The response travels back but is lost in transit (packet loss).
# The client receives a TimeoutException and retries.

asyncio.run(place_order("customer_001", "item_abc", 9999))
```

**Questions:**

1. **Predict the outcome.** After both attempts complete, how many orders exist in the database? How many times has the customer been charged?

2. **Explain the mechanism.** The developer argues: "I only called `place_order` once. The server is stateless — how can there be two orders?" Walk through what happened at the network level, step by step, explaining why statelessness makes this *worse*, not better.

3. **Identify the property.** What specific HTTP property (use the exact term from the lecture) does POST lack that makes this retry dangerous? List the three methods that would be safe to retry without an idempotency key in this scenario.

4. **Propose a solution.** Without writing full implementation code, describe the exact mechanism (covered in Part 3.6 of the lecture) that allows POST to be made safe to retry. What must the client generate? What must the server do on first call and on duplicate calls?

---

## Exercise 1.3: The CORS That Wasn't Broken

```
Scenario:
A developer's React app is running on http://localhost:3000.
Their FastAPI backend is running on http://localhost:8000.

The browser console shows this error:
───────────────────────────────────────────────────────────────────
Access to fetch at 'http://localhost:8000/users' from origin
'http://localhost:3000' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the
requested resource.
───────────────────────────────────────────────────────────────────

The developer runs two terminal tests to confirm the server is broken:

Test 1 — Plain curl:
$ curl http://localhost:8000/users
[{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}]   ← Works!

Test 2 — curl with explicit Origin header (simulating a cross-origin request):
$ curl -H "Origin: http://localhost:3000" http://localhost:8000/users
[{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}]   ← Still works!

Their conclusion: "curl works, browser fails — the server must be
selectively blocking the browser somehow."
```

**Questions:**

1. **Is the developer's conclusion correct?** They claim "the server is blocking the browser request." Evaluate this precisely: did the server receive the browser's request? Did the server send a response? What entity is actually doing the blocking?

2. **Predict the server logs.** When the browser makes the blocked request, does an entry appear in the FastAPI access log? State yes or no and explain exactly why.

3. **Explain the curl discrepancy.** Test 2 explicitly includes `Origin: http://localhost:3000` — the exact origin that the browser is being blocked for. Yet curl succeeds. Why doesn't curl experience the same CORS error that the browser does?

4. **Define the protection.** If CORS doesn't protect the server, what *is* it protecting? Be specific about who the subject is and what attack it prevents.

---

## Exercise 1.4: Reading the Server's Vocabulary

A developer is building an API client and receives these six responses in sequence. For each one, answer the diagnostic questions below.

```python
responses = [
    # Response 1
    {
        "status": 401,
        "headers": {"WWW-Authenticate": "Bearer"},
        "body": {"error": {"code": "NO_CREDENTIALS", "message": "Authorization header missing"}}
    },

    # Response 2
    {
        "status": 403,
        "body": {"error": {"code": "INSUFFICIENT_ROLE", "message": "Admin role required for this endpoint"}}
    },

    # Response 3
    {
        "status": 404,
        "body": {"error": {"code": "RESOURCE_NOT_FOUND", "message": "Book with id=9999 does not exist"}}
    },

    # Response 4
    {
        "status": 409,
        "body": {"error": {"code": "DUPLICATE_ISBN", "message": "A book with ISBN 978-0-13-468599-1 already exists"}}
    },

    # Response 5
    {
        "status": 500,
        "body": {
            "error": {
                "code": "INTERNAL_ERROR",
                "message": "An unexpected error occurred. Please try again.",
                "details": []
            }
        }
    },

    # Response 6
    {
        "status": 422,
        "body": {"error": {"code": "VALIDATION_ERROR", "message": "Request validation failed",
                           "details": [{"field": "year", "message": "Must be between 1000 and 2026"}]}}
    }
]
```

**Questions — apply to each response:**

1. Which family (1xx/2xx/3xx/4xx/5xx) does this belong to? What is its general meaning?
2. Is the fault on the client side, the server side, or both?
3. Will retrying the exact same request resolve the error? Explain why or why not.

**Special questions:**

4. **Response 5 only.** A colleague proposes: "We should add the full Python stack trace and the raw SQL query to the 500 response body so API consumers can debug what went wrong on the server side." Evaluate this suggestion and its security implications.

5. **Responses 1 and 2 only.** Using the library front desk analogy from the lecture, explain in concrete terms the difference between these two responses. What must the client do differently to fix each one?

---

# Level 1 Solutions

## Solution 1.1: The Vanishing Book

**1. PUT result — the book record immediately after the PUT:**
```json
{
  "id": 42,
  "author": "Robert C. Martin"
}
```
`title`, `year`, and `available` are gone. PUT replaced the entire resource with exactly what was sent in the body. The server interpreted the request as: *"The new complete state of book #42 is `{"author": "Robert C. Martin"}` — nothing else."* Fields not included are absent from the new state, not preserved.

**2. PATCH result — applied to the already-destroyed record:**
```json
{
  "id": 42,
  "author": "Robert C. Martin",
  "available": true
}
```
PATCH applied `{"available": true}` to the existing record — which at that point was already missing `title` and `year` from the PUT. The PATCH only modifies what it mentions; it cannot restore fields that were already deleted. The damage from the PUT was not undone.

**3. Why PUT is not like PATCH:**
PUT has *replace* semantics: "the body I am sending is the complete new state of this resource." It does not have *merge* semantics. When you send `{"author": "Robert C. Martin"}`, the server does not read "update author." It reads "this is everything this resource now contains." Fields absent from the body are not "unchanged" — they are *not present in the new state* and are therefore deleted.

PATCH has *partial update* semantics: "apply only these changes, leave everything else untouched." Fields not mentioned are explicitly preserved.

The junior developer's model assumes PUT "merges" the incoming data with the existing record. PUT does not. It replaces. This is why most production systems prefer PATCH for updates — the risk of accidental full replacement is eliminated.

**4. Production consequence:**
`title` (`"Clean Code"`) and `year` (`2008`) were permanently deleted from the database record for book #42 by a routine "author name fix." Recovery requires restoring from a database backup or replaying from an audit log — neither of which is fast or simple. This class of bug frequently goes undetected until a customer reports that their data is missing.

---

## Solution 1.2: The Retry Loop

**1. Outcome:**
Two orders exist in the database. The customer has been charged twice. Order #99 (from Attempt 1) and Order #100 (from Attempt 2) are both valid records.

**2. Step-by-step network mechanism:**
```
Attempt 1:
1. Client sends POST /orders {"customer_id": "customer_001", ...}
2. Server receives it — stateless, processes fresh
3. Server creates Order #99 and charges the card
4. Server sends "HTTP/1.1 201 Created" back through the network
5. A packet is lost in transit — the response never arrives at the client
6. Client's httpx timeout fires: TimeoutException
7. Client concludes: "The request must have failed." (WRONG — it succeeded)

Attempt 2:
8. Client sends the identical POST /orders request
9. Server receives it — stateless, so it has NO RECORD of Attempt 1
   (This is exactly why statelessness makes it worse here — the server
    cannot check "did I already handle this?" without extra infrastructure)
10. Server creates Order #100 and charges the card again
11. Server sends 201. Client receives it this time.
```

Statelessness means the server processes each request in isolation. It is not a bug — it is the design. The bug is using POST retry without idempotency infrastructure.

**3. The property:**
POST is **not idempotent**. Performing the same POST twice produces a different result than performing it once (two resources instead of one). The three safe methods to retry without an idempotency key:
- **GET** — safe and idempotent: retrieves the same data repeatedly
- **PUT** — idempotent: setting a resource to the same state N times leaves it in that state
- **DELETE** — idempotent: the resource is absent after one call; additional calls attempt to delete something already gone (server returns 204 or 404, resource remains absent)

**4. The solution — idempotency keys:**
Before the first attempt, the client generates a UUID: `idempotency_key = str(uuid.uuid4())`. This key is included as a header on the request: `Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000`. On first receipt, the server processes the request normally and stores the result in Redis, keyed by the UUID. On any subsequent request with the same key, the server detects it has already processed this operation and returns the *stored result* — no new order, no new charge. The client receives a successful response either way, and the database contains exactly one order.

---

## Solution 1.3: The CORS Confusion

**1. The developer's conclusion is wrong.**
The server received the browser's request. The server processed it. The server sent back a valid 200 OK response containing the user data. The block occurred *after the response left the server and arrived at the browser*. The browser inspected the response headers, found no `Access-Control-Allow-Origin` header matching `http://localhost:3000`, and refused to expose the response body to the JavaScript code that made the fetch call. The server was unaware anything went wrong.

**2. Server logs:**
**Yes**, an entry appears. You will see something like:
```
INFO: 127.0.0.1:52341 - "GET /users HTTP/1.1" 200 OK
```
The server logged a successful 200 response. From the server's perspective, the request was handled correctly. The CORS block happened on the client side, after the response was delivered.

**3. Curl discrepancy:**
curl is not a browser. It has no concept of "the page's origin," no Same-Origin Policy, and no mechanism for enforcing CORS. curl sends an HTTP request and displays whatever the server returns — unconditionally, regardless of what `Origin` header is included. CORS is a browser security feature implemented inside the browser engine. It is not a server feature, not a network feature, and not enforced by any tool that is not a browser.

**4. What CORS actually protects:**
CORS protects the *browser user* from cross-site request forgery via JavaScript. Without CORS, a malicious website could include JavaScript that silently calls your API using the victim's stored browser cookies or session tokens — without the user's knowledge or consent. The victim visits `evil.com`; evil.com's JavaScript calls `https://api.mybank.com/transfer` using the victim's active banking session. CORS prevents evil.com's JavaScript from reading the API's response, and the browser refuses to send credentials cross-origin unless the server explicitly grants permission. The attacker is prevented from conducting the operation covertly in the background.

---

## Solution 1.4: Status Code Diagnostics

| Response | Family | Fault | Retry same request? |
|---|---|---|---|
| 401 | 4xx Client Error | Client | ❌ No — must provide valid credentials |
| 403 | 4xx Client Error | Client | ❌ No — must obtain elevated permissions |
| 404 | 4xx Client Error | Client | ❌ No — resource does not exist; fix the ID or create the resource first |
| 409 | 4xx Client Error | Client | ❌ No — the conflict (duplicate ISBN) will not resolve itself |
| 500 | 5xx Server Error | Server | ✅ Maybe — wait briefly, retry once; report the bug to the server team |
| 422 | 4xx Client Error | Client | ❌ No — fix the `year` field to be within the valid range |

**Response 5 — the stack trace suggestion:**
This must never be done. Exposing database exception messages, SQL query text, table names, column names, or Python tracebacks in a 5xx response body reveals the server's internal architecture to potential attackers. It tells them precisely: which database engine is in use, what tables and columns exist, what queries are being executed, and where code paths break. This information directly enables SQL injection targeting, schema enumeration, and denial-of-service attacks crafted to exploit known failure modes. Internal error details must be captured in server-side logs — visible only to operators — and never surfaced in the client-facing response. The client receives only a stable, generic message.

**Responses 1 and 2 — library analogy:**

- **401 — "Who are you?":** You approach the library's restricted archive room. You have no library card whatsoever. The librarian says: *"I cannot let you in because I have no idea who you are. Please show me your card."* The fix is to authenticate — produce valid credentials. The problem is identity, not permission.

- **403 — "I know who you are, but no.":** You approach the same room with a valid basic-tier library card. The librarian scans it, confirms exactly who you are, and says: *"I know you perfectly well — your basic card does not permit access to the restricted archive. No."* Showing the card more firmly or showing it a different way will not help. The fix is to obtain a higher-tier card — elevated permissions. The problem is authorisation level, not identity.

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Cache That Serves Lies

A developer built a simple in-memory cache for the book availability endpoint. The code runs without errors and appears to work. Something is wrong.

```python
import httpx
import asyncio
from typing import Optional

BASE_URL = "http://api.library.com"

# In-memory cache: key → book dict
cache: dict[str, dict] = {}

async def get_book(book_id: int) -> Optional[dict]:
    """Fetch a book, using cache if available."""
    cache_key = f"book_{book_id}"

    if cache_key in cache:
        print(f"[CACHE HIT] Returning cached book {book_id}")
        return cache[cache_key]

    async with httpx.AsyncClient() as client:
        response = await client.get(f"{BASE_URL}/books/{book_id}")

        if response.status_code == 200:
            data = response.json()
            cache[cache_key] = data
            print(f"[CACHE MISS] Fetched and cached book {book_id}")
            return data

        return None

async def update_book_availability(book_id: int, available: bool) -> dict:
    """Update a book's availability status via PATCH."""
    async with httpx.AsyncClient() as client:
        response = await client.patch(
            f"{BASE_URL}/books/{book_id}",
            json={"available": available}
        )
        return response.json()

async def main():
    # Step 1: Librarian checks availability (cache miss → fetches from API)
    book = await get_book(42)
    print(f"Is available: {book['available']}")   # True

    # Step 2: A student borrows the book (marks it unavailable)
    await update_book_availability(42, available=False)
    print("Book checked out by student.")

    # Step 3: A second librarian checks availability 30 seconds later
    book = await get_book(42)
    print(f"Is available: {book['available']}")   # What does this print?

asyncio.run(main())
```

---

**Task 2a: Identify the Flaw**

1. What does Step 3 print for `book['available']`? Is this correct?
2. What is the specific flaw in the caching logic?
3. Which HTTP concept from Part 5.4 of the lecture is this a violation of?

---

**Task 2b: Explain the Mechanism**

The PATCH in Step 2 succeeded — the API server's database record is correct (`available: false`). Trace the execution path of Step 3's `get_book(42)` call line by line, and explain exactly why the wrong value is returned.

---

**Task 2c: Instrument to Prove Diagnosis**

Add logging inside `get_book` to prove the stale cached value is being returned. Show the complete console output — including the cached value — that exposes the bug:

```python
async def get_book(book_id: int) -> Optional[dict]:
    cache_key = f"book_{book_id}"

    if cache_key in cache:
        # Add instrumentation here to show:
        # 1. That the cache branch is being taken
        # 2. The exact cached value being returned
        # 3. A warning that this may be stale
        ...
```

---

**Task 2d: Fix the Code**

Fix `update_book_availability` so it invalidates the relevant cache entry after a successful update. Your fix must:
- Invalidate only the cache entry for the book that was updated (not the entire cache)
- Only invalidate on a successful HTTP response (not on network failure)
- Not break the caching behaviour for books that were not updated

```python
async def update_book_availability(book_id: int, available: bool) -> dict:
    # Your fix here
    ...
```

---

**Task 2e: Write a Verification Test**

Write a test that fails on the original broken code and passes after your fix:

```python
async def test_cache_invalidated_after_update():
    """
    After updating a resource, subsequent reads must not serve stale data.
    This test FAILS on the original code and PASSES on the fixed version.
    """
    # Your test here — hint: prime the cache, update, assert the cache entry is gone
    ...
```

---

## Exercise 2.2: The API That Always Says 200

A developer built three FastAPI endpoints. They run without crashing. However, they violate HTTP semantics in three distinct ways.

```python
from fastapi import FastAPI

app = FastAPI()

books: dict[int, dict] = {
    1: {"id": 1, "title": "Clean Code", "author": "Robert Martin", "available": True},
    2: {"id": 2, "title": "Design Patterns", "author": "Gang of Four", "available": True},
}

@app.get("/books/{book_id}")
async def get_book(book_id: int):
    try:
        book = books[book_id]
        return book
    except Exception:
        return {"error": "Something went wrong", "book_id": book_id}

@app.post("/books")
async def create_book(book_data: dict):
    try:
        new_id = max(books.keys()) + 1
        book = {"id": new_id, **book_data}
        books[new_id] = book
        return book
    except Exception:
        return {"error": "Could not create book"}

@app.delete("/books/{book_id}")
async def delete_book(book_id: int):
    try:
        del books[book_id]
        return {"message": "Deleted successfully"}
    except KeyError:
        return {"error": "Book not found"}
```

---

**Task 2a: Identify the Three Flaws**

Without running the code, identify the HTTP semantic violation in each endpoint:

1. What status code does `GET /books/999` (a book that does not exist) return? What should it return?
2. What status code does `POST /books` (a successful creation) return? What should it return?
3. What status code does `DELETE /books/1` (a successful deletion) return? What should it return, and what must the response body contain?

---

**Task 2b: Explain the Consequences**

For each flaw, describe one concrete failure scenario that would occur in a client that correctly follows HTTP conventions:

1. A React frontend that checks `if (response.status === 404)` to show a "not found" message.
2. A mobile app that checks `if (response.status === 201)` to navigate to the newly created resource.
3. An automated test suite that asserts `assert response.body == ""` after a delete operation.

---

**Task 2c: Instrument to Prove Diagnosis**

Complete and run this verification block to prove the actual status codes:

```python
import httpx
from fastapi.testclient import TestClient

client = TestClient(app)

r1 = client.get("/books/999")
print(f"GET nonexistent book:  {r1.status_code}")   # Should be 404, is ___?

r2 = client.post("/books", json={"title": "New Book", "author": "Alice"})
print(f"POST create book:      {r2.status_code}")   # Should be 201, is ___?

r3 = client.delete("/books/1")
print(f"DELETE existing book:  {r3.status_code}")   # Should be 204, is ___?
print(f"DELETE body:           {r3.text!r}")         # Should be '', is ___?
```

---

**Task 2d: Fix the Code**

Rewrite all three endpoints with correct HTTP semantics, using `HTTPException` for errors and the error response structure from Part 4.6 of the lecture. Enforce correct status codes for both success and failure cases.

---

**Task 2e: Write Verification Tests**

Write tests that fail on the original code and pass after your fix:

```python
def test_get_nonexistent_book_returns_404():
    ...

def test_create_book_returns_201():
    ...

def test_delete_returns_204_with_empty_body():
    ...
```

---

# Level 2 Solutions

## Solution 2.1: The Cache That Serves Lies

**Task 2a:**

1. Step 3 prints `Is available: True`. This is **wrong**. The book is unavailable — the PATCH succeeded and the database record is correct — but the stale cache is served.
2. The cache is populated on read but **never invalidated on write**. When `update_book_availability` PATCHes the server, it makes no change to the in-memory cache dict. The next `get_book(42)` finds `book_42` in the cache and returns the pre-PATCH value.
3. This violates **cache invalidation** — the principle that a write operation must invalidate or update any cache entries for the affected resource. In the HTTP caching layer (Section 5.4), this is handled by `Cache-Control` directives, ETags, and conditional requests. Our in-memory dict has none of these mechanisms.

**Task 2b:** Execution trace for Step 3's `get_book(42)`:
```
1. cache_key = "book_42"
2. "book_42" IN cache? → YES (populated during Step 1's cache miss)
3. print("[CACHE HIT] Returning cached book 42")
4. return cache["book_42"]
   ↑ This is the dict object stored during Step 1.
     It has available=True.
     The PATCH in Step 2 updated the API server's database.
     It did NOT modify cache["book_42"].
     The two are now out of sync.
```

**Task 2c:**
```python
async def get_book(book_id: int) -> Optional[dict]:
    cache_key = f"book_{book_id}"

    if cache_key in cache:
        cached_value = cache[cache_key]
        print(f"[CACHE HIT] Returning cached book {book_id}")
        print(f"[CACHE HIT] Cached 'available' value: {cached_value.get('available')}")
        print(f"[CACHE WARNING] This value may be stale if the resource was recently modified")
        return cached_value

    # ... rest unchanged
```

Console output proving the bug:
```
[CACHE MISS] Fetched and cached book 42
Is available: True
Book checked out by student.
[CACHE HIT] Returning cached book 42
[CACHE HIT] Cached 'available' value: True    ← STALE — server has False
[CACHE WARNING] This value may be stale if the resource was recently modified
Is available: True                             ← WRONG
```

**Task 2d:**
```python
async def update_book_availability(book_id: int, available: bool) -> dict:
    """Update a book's availability status. Invalidates cache on success."""
    async with httpx.AsyncClient() as client:
        response = await client.patch(
            f"{BASE_URL}/books/{book_id}",
            json={"available": available}
        )

        # Only invalidate on confirmed success — not on network errors
        if response.status_code == 200:
            cache_key = f"book_{book_id}"
            if cache_key in cache:
                del cache[cache_key]
                print(f"[CACHE INVALIDATED] {cache_key}")

        return response.json()
```

**Task 2e:**
```python
async def test_cache_invalidated_after_update():
    """
    After updating a resource, subsequent reads must not serve stale data.
    Fails on original code. Passes on fixed code.
    """
    # Step 1: Prime the cache with a known value
    book = await get_book(42)
    assert book is not None

    cache_key = "book_42"
    assert cache_key in cache, "Cache should be primed after first fetch"

    original_availability = cache[cache_key]["available"]

    # Step 2: Update the resource (flip availability)
    await update_book_availability(42, available=(not original_availability))

    # Step 3: Cache entry for this book must be gone
    assert cache_key not in cache, (
        f"Cache was NOT invalidated after update. "
        f"Still serving stale value: {cache.get(cache_key)}"
    )

    print("PASSED: Cache correctly invalidated after write.")
```

---

## Solution 2.2: The API That Always Says 200

**Task 2a:**
1. `GET /books/999` returns **200 OK** with `{"error": "Something went wrong", "book_id": 999}`. Should return **404 Not Found**.
2. `POST /books` returns **200 OK** with the new book data. Should return **201 Created**.
3. `DELETE /books/1` returns **200 OK** with `{"message": "Deleted successfully"}`. Should return **204 No Content** with an **empty body**.

**Task 2b:**
1. **React frontend:** `if (response.status === 404)` never fires. The frontend receives 200 with an error object in the body, tries to render it as a book, and crashes when accessing `book.title` which is undefined. The user sees a blank page or a JavaScript error instead of a "not found" message.

2. **Mobile app:** `if (response.status === 201)` never fires. The app never navigates to the new resource. The user sees their create action appear to do nothing. If they tap "Create" again, a duplicate resource is created.

3. **Automated test:** `assert response.body == ""` fails because the body contains `{"message": "Deleted successfully"}`. The test suite marks delete operations as broken when they are actually working correctly.

**Task 2c:** Verified output:
```
GET nonexistent book:  200   ← Should be 404
POST create book:      200   ← Should be 201
DELETE existing book:  200   ← Should be 204
DELETE body:           '{"message":"Deleted successfully"}'  ← Should be ''
```

**Task 2d:**
```python
from fastapi import FastAPI, HTTPException
from fastapi.responses import JSONResponse, Response

app = FastAPI()

books: dict[int, dict] = {
    1: {"id": 1, "title": "Clean Code", "author": "Robert Martin", "available": True},
    2: {"id": 2, "title": "Design Patterns", "author": "Gang of Four", "available": True},
}

@app.get("/books/{book_id}")
async def get_book(book_id: int):
    if book_id not in books:
        raise HTTPException(
            status_code=404,
            detail={
                "error": {
                    "code": "RESOURCE_NOT_FOUND",
                    "message": f"Book with id={book_id} does not exist",
                    "details": []
                }
            }
        )
    return books[book_id]   # 200 OK (FastAPI default)

@app.post("/books", status_code=201)
async def create_book(book_data: dict):
    new_id = max(books.keys()) + 1
    book = {"id": new_id, **book_data}
    books[new_id] = book
    # Explicit 201 — FastAPI does not default to this for POST
    return JSONResponse(content=book, status_code=201)

@app.delete("/books/{book_id}", status_code=204)
async def delete_book(book_id: int):
    if book_id not in books:
        raise HTTPException(
            status_code=404,
            detail={
                "error": {
                    "code": "RESOURCE_NOT_FOUND",
                    "message": f"Book with id={book_id} does not exist",
                    "details": []
                }
            }
        )
    del books[book_id]
    return Response(status_code=204)  # Empty body — required by spec
```

**Task 2e:**
```python
from fastapi.testclient import TestClient

client = TestClient(app)

def test_get_nonexistent_book_returns_404():
    r = client.get("/books/999")
    assert r.status_code == 404, f"Expected 404, got {r.status_code}"
    body = r.json()
    assert body["error"]["code"] == "RESOURCE_NOT_FOUND"

def test_create_book_returns_201():
    r = client.post("/books", json={"title": "Refactoring", "author": "Fowler"})
    assert r.status_code == 201, f"Expected 201, got {r.status_code}"
    assert "id" in r.json()

def test_delete_returns_204_with_empty_body():
    r = client.delete("/books/1")
    assert r.status_code == 204, f"Expected 204, got {r.status_code}"
    assert r.text == "", f"Expected empty body, got: {r.text!r}"
```

---

# LEVEL 2.5: EXTEND EXISTING CODE

## Exercise 2.5: The HTTP Client That Needs to Grow Up

The following code is correct and functional. These tests currently pass:

```python
import httpx
import asyncio
from typing import Optional

BASE_URL = "http://api.library.com"

async def fetch_book(book_id: int) -> Optional[dict]:
    """Fetch a single book from the library API."""
    async with httpx.AsyncClient() as client:
        response = await client.get(f"{BASE_URL}/books/{book_id}")
        if response.status_code == 200:
            return response.json()
        return None

async def create_book(title: str, author: str, year: int) -> dict:
    """Create a new book. Returns the created book with its server-assigned ID."""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{BASE_URL}/books",
            json={"title": title, "author": author, "year": year}
        )
        return response.json()

async def delete_book(book_id: int) -> bool:
    """Delete a book. Returns True if deleted, False if not found."""
    async with httpx.AsyncClient() as client:
        response = await client.delete(f"{BASE_URL}/books/{book_id}")
        return response.status_code == 204
```

```python
# These tests must STILL PASS after every extension

async def test_fetch_existing_book():
    book = await fetch_book(42)
    assert book is not None
    assert book["id"] == 42

async def test_fetch_missing_book():
    book = await fetch_book(99999)
    assert book is None

async def test_create_book():
    book = await create_book("Refactoring", "Martin Fowler", 2018)
    assert "id" in book
    assert book["title"] == "Refactoring"

async def test_delete_existing_book():
    result = await delete_book(42)
    assert result is True

async def test_delete_missing_book():
    result = await delete_book(99999)
    assert result is False
```

---

**Extension 2.5.1: Rate Limit Awareness**

The API is behind a rate limiter. When you exceed your quota, it returns `429 Too Many Requests` with a `Retry-After` header containing the seconds to wait.

Extend `fetch_book` to:
- Detect a 429 response
- Read the `Retry-After` header (default to 60 seconds if the header is absent)
- Wait the specified duration **without blocking the event loop**
- Retry exactly once after waiting
- Return `None` if the retry also fails

**Constraint:** You may not use `time.sleep()`. Explain why using it would be a critical mistake in an async context.

All five existing tests must still pass.

---

**Extension 2.5.2: Idempotency Key for create_book**

`create_book` is currently vulnerable to double-creation if a network timeout triggers a retry. Extend it to:
- Accept an optional parameter `idempotency_key: Optional[str] = None`
- If a key is provided, send it as an `Idempotency-Key` header
- If no key is provided, **generate a UUID4 automatically** — so every call is safe by default without the caller needing to think about it
- Return the same dict shape regardless of whether a key was provided

All five existing tests must still pass.

---

**Extension 2.5.3: Shared Connection Pool**

Each function currently opens a brand-new `httpx.AsyncClient` per call. For a script making 200 API calls, this means 200 TCP handshakes and — if using HTTPS — 200 TLS negotiations.

Refactor the module to:
- Expose a `LibraryClient` class that owns a single shared `httpx.AsyncClient`
- Support an async context manager: `async with LibraryClient() as client:`
- Move all three operations to methods on the class
- The underlying HTTP client must be created on `__aenter__` and properly closed on `__aexit__`

State the concrete reason why a shared client is faster than one-per-call, using the terms from Part 2.3 of the lecture.

---

# Level 2.5 Solutions

## Solution 2.5.1: Rate Limit Awareness

```python
import asyncio

async def fetch_book(book_id: int) -> Optional[dict]:
    """Fetch a book. Handles 429 with Retry-After before one retry."""
    async with httpx.AsyncClient() as client:
        response = await client.get(f"{BASE_URL}/books/{book_id}")

        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", "60"))
            print(f"Rate limited. Waiting {retry_after}s before retry...")
            await asyncio.sleep(retry_after)   # ← Non-blocking
            response = await client.get(f"{BASE_URL}/books/{book_id}")

        if response.status_code == 200:
            return response.json()
        return None
```

**Why `time.sleep()` is forbidden here:**
`time.sleep()` is a blocking call — it suspends the **entire OS thread**, which means the entire asyncio event loop is frozen for that duration. No other coroutines can run. No other HTTP requests can be served. In a FastAPI server handling concurrent requests, a single `time.sleep(60)` in an async endpoint would freeze the server for 60 seconds for every client. `await asyncio.sleep()` surrenders control back to the event loop, allowing other coroutines to run while this one waits. This distinction is covered in Week 1, Lecture 3.

---

## Solution 2.5.2: Idempotency Key

```python
import uuid

async def create_book(
    title: str,
    author: str,
    year: int,
    idempotency_key: Optional[str] = None,
) -> dict:
    """Create a new book. Idempotency key prevents double-creation on retry."""

    # Auto-generate if not provided — safe by default
    if idempotency_key is None:
        idempotency_key = str(uuid.uuid4())

    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{BASE_URL}/books",
            json={"title": title, "author": author, "year": year},
            headers={"Idempotency-Key": idempotency_key},
        )
        return response.json()
```

**Why auto-generate when no key is provided:** If the caller must remember to generate a UUID themselves, they will sometimes forget and the function is unsafe in those cases. Generating automatically means `create_book` is always safe to retry — the caller just passes the same key on retry. Callers who don't care about retry safety can ignore the parameter entirely; it is silently handled.

---

## Solution 2.5.3: Shared Connection Pool

```python
class LibraryClient:
    """
    HTTP client for the Library API.
    Shares a single connection pool across all operations.
    """

    def __init__(self):
        self._client: Optional[httpx.AsyncClient] = None

    async def __aenter__(self) -> "LibraryClient":
        self._client = httpx.AsyncClient(base_url=BASE_URL)
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self._client:
            await self._client.aclose()
        return False   # Do not suppress exceptions

    async def fetch_book(self, book_id: int) -> Optional[dict]:
        response = await self._client.get(f"/books/{book_id}")
        if response.status_code == 200:
            return response.json()
        return None

    async def create_book(self, title: str, author: str, year: int,
                          idempotency_key: Optional[str] = None) -> dict:
        if idempotency_key is None:
            idempotency_key = str(uuid.uuid4())
        response = await self._client.post(
            "/books",
            json={"title": title, "author": author, "year": year},
            headers={"Idempotency-Key": idempotency_key},
        )
        return response.json()

    async def delete_book(self, book_id: int) -> bool:
        response = await self._client.delete(f"/books/{book_id}")
        return response.status_code == 204


# Usage — one pool, all operations:
async def main():
    async with LibraryClient() as client:
        book   = await client.fetch_book(42)
        new    = await client.create_book("Refactoring", "Fowler", 2018)
        result = await client.delete_book(42)
```

**Why a shared client is faster:**
As established in Part 2.3, opening a new `httpx.AsyncClient` per call performs a full **TCP handshake** (and TLS negotiation for HTTPS) on every single request. A `LibraryClient` maintains a **persistent connection pool** internally. Subsequent requests reuse existing TCP connections — no handshake, no TLS negotiation. For 200 sequential API calls, this eliminates 199 handshakes. The latency saving per call is typically 10–80ms depending on network conditions.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## Exercise 3.1: The Idempotent Checkout System

**Scenario:**
A university library has a student checkout API. Students on campus Wi-Fi frequently experience timeouts when submitting checkout requests. They click "Check Out" multiple times when they don't see a confirmation, resulting in duplicate checkout records — and, worse, the same book being marked as borrowed twice.

Your job is to design `POST /checkouts` so that:
- Retrying a checkout request with the same idempotency key never creates a duplicate checkout
- Two genuinely different checkout requests (even with identical payloads) create two records if they use different keys
- The same key sent with a different body is explicitly rejected
- The `Idempotency-Key` header is required — its absence is a client error

**Constraint:**
You may NOT solve this using a business rule like "check if this student already has this book checked out." That is a domain constraint, not idempotency. Idempotency must be key-based.

---

**Provided tests (all must pass — do not modify them):**

```python
import pytest
import uuid
from fastapi.testclient import TestClient

# client fixture assumed — wraps your FastAPI app

def test_checkout_returns_201(client):
    key = str(uuid.uuid4())
    response = client.post(
        "/checkouts",
        json={"student_id": "s001", "book_id": 42},
        headers={"Idempotency-Key": key}
    )
    assert response.status_code == 201
    assert "checkout_id" in response.json()

def test_retry_with_same_key_returns_same_result(client):
    """Same key + same body → same checkout_id, no new record."""
    key = str(uuid.uuid4())
    payload = {"student_id": "s001", "book_id": 42}

    r1 = client.post("/checkouts", json=payload, headers={"Idempotency-Key": key})
    r2 = client.post("/checkouts", json=payload, headers={"Idempotency-Key": key})

    assert r1.status_code == 201
    assert r2.status_code == 200           # Not 201 — not newly created
    assert r1.json()["checkout_id"] == r2.json()["checkout_id"]

def test_different_keys_create_different_checkouts(client):
    """Different keys → two separate checkouts (even with same payload)."""
    payload = {"student_id": "s001", "book_id": 43}

    r1 = client.post("/checkouts", json=payload, headers={"Idempotency-Key": str(uuid.uuid4())})
    r2 = client.post("/checkouts", json=payload, headers={"Idempotency-Key": str(uuid.uuid4())})

    assert r1.status_code == 201
    assert r2.status_code == 201
    assert r1.json()["checkout_id"] != r2.json()["checkout_id"]

def test_same_key_different_body_returns_409(client):
    """Same key, different body → conflict."""
    key = str(uuid.uuid4())

    r1 = client.post("/checkouts",
        json={"student_id": "s001", "book_id": 42},
        headers={"Idempotency-Key": key}
    )
    r2 = client.post("/checkouts",
        json={"student_id": "s001", "book_id": 99},  # Different book!
        headers={"Idempotency-Key": key}             # Same key!
    )

    assert r1.status_code == 201
    assert r2.status_code == 409

def test_missing_key_returns_400(client):
    """No Idempotency-Key header → client error."""
    response = client.post(
        "/checkouts",
        json={"student_id": "s001", "book_id": 42}
        # No Idempotency-Key header
    )
    assert response.status_code == 400
    assert "Idempotency-Key" in response.json()["error"]["message"]
```

---

**Task 3a: Implement the Core**

Implement `POST /checkouts` that passes `test_checkout_returns_201` and `test_retry_with_same_key_returns_same_result`. You may use a plain dict as your idempotency store for now.

---

**Task 3b: Full Contract**

Extend your implementation to pass all five tests. This requires detecting body mismatches for the same key. Describe how you will fingerprint the request body to detect changes.

---

**Task 3c: Justify Two Design Decisions**

Answer in writing:
1. The retry response uses 200, not 201. Why? Would returning 201 again be semantically wrong?
2. The same key with a different body returns 409 Conflict rather than silently using the stored result. Why is silently returning the stored result dangerous?

---

# Level 3 Solutions

## Solution 3.1

**Task 3a: Core Implementation**

```python
from fastapi import FastAPI, Header, HTTPException
from fastapi.responses import JSONResponse
from pydantic import BaseModel
from typing import Optional
import hashlib, json, uuid

app = FastAPI()

# Production equivalent: Redis with TTL expiry
idempotency_store: dict[str, dict] = {}
checkouts: dict[str, dict] = {}

class CheckoutRequest(BaseModel):
    student_id: str
    book_id: int

def _body_fingerprint(data: dict) -> str:
    """SHA-256 of the canonically serialised body."""
    return hashlib.sha256(
        json.dumps(data, sort_keys=True).encode()
    ).hexdigest()
```

**Task 3b: Full Implementation**

```python
@app.post("/checkouts")
async def create_checkout(
    request: CheckoutRequest,
    idempotency_key: Optional[str] = Header(None, alias="Idempotency-Key"),
):
    # Reject missing key before any processing
    if idempotency_key is None:
        raise HTTPException(
            status_code=400,
            detail={
                "error": {
                    "code": "MISSING_HEADER",
                    "message": "Idempotency-Key header is required for this endpoint",
                    "details": [{"field": "Idempotency-Key", "message": "Header is missing"}]
                }
            }
        )

    incoming_fingerprint = _body_fingerprint(request.model_dump())

    if idempotency_key in idempotency_store:
        stored = idempotency_store[idempotency_key]

        # Same key, different body → conflict
        if stored["fingerprint"] != incoming_fingerprint:
            raise HTTPException(
                status_code=409,
                detail={
                    "error": {
                        "code": "IDEMPOTENCY_KEY_REUSE",
                        "message": "Idempotency-Key was already used with a different request body",
                        "details": []
                    }
                }
            )

        # Same key, same body → return stored result (200, not 201)
        return JSONResponse(content=stored["result"], status_code=200)

    # First time — process and store
    checkout_id = str(uuid.uuid4())
    checkout = {
        "checkout_id": checkout_id,
        "student_id": request.student_id,
        "book_id": request.book_id,
        "status": "active",
    }
    checkouts[checkout_id] = checkout
    idempotency_store[idempotency_key] = {
        "result": checkout,
        "fingerprint": incoming_fingerprint,
    }

    return JSONResponse(content=checkout, status_code=201)
```

**Task 3c: Design Justifications**

**1. Retry returns 200, not 201:**
201 means "a new resource was created as a result of this request." On a retry, no new resource was created — the checkout already existed. Returning 201 again would be a lie. A client that checks `if (status === 201) { navigate_to_new_resource() }` would behave correctly with 200 on retry: it already has the checkout data from the first 201 and can handle 200 as a confirmation. The status code must truthfully describe what happened, not merely whether the operation "succeeded."

**2. Body mismatch returns 409, not the stored result:**
Silently returning the stored result when the body changed is dangerous because it hides a logic error in the client. Consider: a payment service generates a key for a $10 charge, sends it successfully, but then generates a bug that reuses the same key for a $500 charge. If the server silently returns the $10 result, the client believes the $500 charge succeeded when it was never processed. The discrepancy between what was intended and what actually happened is invisible. Rejecting with 409 forces the client to surface this as an explicit error, enabling the bug to be detected and fixed rather than silently producing wrong outcomes.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## Exercise 4: When REST's Rules Make Things Worse

The lecture establishes REST's Uniform Interface: "Resources are nouns, methods are verbs. Use GET/POST/PUT/PATCH/DELETE." But this principle, applied rigidly, can produce APIs that are harder to use, more dangerous, and less semantically honest than alternatives.

---

**4a: The State Machine Problem**

Consider this endpoint designed to "publish" a draft article — a multi-step orchestrated operation:

```python
# The "pure REST" approach
# PATCH /articles/42 {"status": "published"}

# What the server does when it receives this:
async def handle_publish_patch(article_id: int):
    # Step 1: Update database (sets status = "published") ✓
    await db.execute("UPDATE articles SET status='published' WHERE id=?", article_id)
    
    # Step 2: Send email to 10,000 newsletter subscribers ✓
    await email_service.send_newsletter(article_id)
    
    # Step 3: Update full-text search index ← NETWORK TIMEOUT HERE
    await search_index.update(article_id)
    
    # Step 4: Invalidate CDN cache ← NEVER REACHED
    await cdn.invalidate(f"/articles/{article_id}")
```

Write a code trace — not a description — showing exactly what happens when a client retries the PATCH after the timeout in Step 3. Include what side effects fire on the retry that should not fire again, and why the PATCH's idempotency property fails to protect against this.

---

**4b: Identify the Class of Operations**

Define the class of operations for which `POST /resource/{id}/action` is more appropriate than `PATCH /resource/{id} {"status": "new_status"}`. What two properties must an operation have to belong to this class? Give three concrete examples from a real-world system, one of which involves money.

---

**4c: Is it Still REST?**

A senior developer says: "Using `POST /articles/42/publish` means this API is not RESTful — it has a verb in the URL." Evaluate this claim using the Richardson Maturity Model. Is a verb-based action endpoint at Level 0, 1, 2, or 3? Is the senior developer's statement accurate?

---

# Level 4 Solutions

## Solution 4.1

**4a: The State Machine Failure Trace**

```
ATTEMPT 1:
1. PATCH /articles/42 {"status": "published"} arrives at server
2. DB UPDATE: articles.status = 'published'                         ← committed
3. Email: 10,000 newsletter emails sent                             ← delivered, irreversible
4. Search index update: network timeout after 5s                    ← FAILED
5. CDN invalidation: never reached                                  ← FAILED
6. Server returns 504 Gateway Timeout (or the exception propagates as 500)
7. Client concludes: "The request failed."

ATTEMPT 2 (client retries the identical PATCH):
8. PATCH /articles/42 {"status": "published"} arrives at server
9. DB UPDATE: articles.status = 'published'
   → Article is already "published". SQL UPDATE is a no-op. ✓ (idempotent)
10. Email: 10,000 newsletter emails sent AGAIN                      ← DUPLICATE
    → Subscribers receive the same newsletter twice.
    → The PATCH was "idempotent" on the database field.
    → It was NOT idempotent on the email side effect.
11. Search index: this time succeeds ✓
12. CDN invalidation: succeeds ✓
13. Server returns 200 OK.

RESULT:
- Database: correct
- Search index: correct
- CDN: correct
- Email subscribers: received 2 notifications for 1 publication event
```

The PATCH's idempotency property guarantees that the *resource state* is the same after N applications as after 1. It makes no guarantee about the idempotency of *side effects* triggered by that state change. When the operation involves irreversible external actions (emails, charges, audit events), PATCH idempotency is insufficient.

**4b: The Class of Operations**

An operation belongs to the "action endpoint" class when it has both of these properties:
1. **Non-idempotent external side effects** — the operation triggers actions in external systems (email, payment, audit log) that must fire exactly once and cannot be safely repeated
2. **State machine semantics** — the operation is only valid in certain current states and must explicitly check that precondition before acting

Three examples:
- `POST /invoices/{id}/send` — sends the invoice to the client via email. Retrying should not re-send. The invoice must be in "draft" state, not "cancelled."
- `POST /payments/{id}/capture` — captures a previously authorised payment. Involves an irreversible charge. Must check the payment is in "authorised" state.
- `POST /orders/{id}/cancel` — cancels an order, triggers a refund, restocks inventory, notifies the warehouse. Multi-system side effects; valid only if status is not "shipped."

**4c: Richardson Maturity Model**

The senior developer's claim is inaccurate. `POST /articles/42/publish` operates at **Level 2** of the Richardson Maturity Model — the same level as the "pure REST" PATCH approach. Level 2 requires using HTTP verbs meaningfully and returning appropriate status codes. It does not prohibit verb-like path segments. Level 3 (HATEOAS) is the level that distinguishes a "truly REST" API.

Roy Fielding's original dissertation defines REST as an architectural style with six constraints (stateless, cacheable, uniform interface, layered system, code on demand, client-server). Nowhere does it prohibit verbs in URL paths — it requires a *uniform interface*, which is about consistency and predictability, not lexical purity. Many mature, well-documented APIs (GitHub: `POST /repos/{owner}/{repo}/actions/runs/{run_id}/cancel`, Stripe: `POST /payment_intents/{id}/confirm`) use action endpoints precisely where they belong.

---

# LEVEL 5: THE TEACH BACK

## Exercise 5.1: "HTTPS Is Just a Padlock Icon"

A junior developer on your team says:

> "HTTPS is basically the same as HTTP. The padlock in the browser is just a visual thing — kind of like a badge. The actual data underneath travels the same way. Our internal microservices API doesn't need HTTPS because it's only called by our own servers inside our private Kubernetes cluster. Adding TLS would just slow things down."

Write a technical correction that covers:
1. Why the "padlock is visual" claim is factually wrong — what TLS actually does to the bytes on the wire
2. A concrete before/after comparison showing what an attacker sees with and without TLS
3. Why "private Kubernetes cluster" does NOT eliminate the need for HTTPS — name two specific attack vectors that remain inside a private network
4. The three distinct guarantees TLS provides, and the concrete damage possible if each one is absent
5. One specific scenario where plain HTTP genuinely is acceptable — and why

---

## Exercise 5.2: "CORS Is My Security"

A junior developer says:

> "I configured CORS to only allow `https://app.mycompany.com`. That means no one outside our company can call our API. I don't need authentication — CORS already filters out unauthorised callers. The API is fully secure."

Write a technical correction that covers:
1. What entity enforces CORS and at what point in the request lifecycle
2. A concrete Python script (under 10 lines) showing how an external attacker trivially bypasses CORS
3. What CORS is actually designed to protect against — name the specific attack and its victim
4. Why CORS and authentication are not alternatives but complements — what each one protects
5. The minimum correct architecture that addresses both concerns

---

## Exercise 5.3: "Stateless Means No User Sessions"

A junior developer says:

> "You told me HTTP APIs are stateless — the server doesn't remember anything between requests. But we need user accounts. Users log in. They have private data. Doesn't that mean we can't be stateless? I think we need to store session objects on the server so it knows who each user is when they make a request."

Write a technical correction that covers:
1. The precise definition of "stateless" — what it forbids and what it explicitly does NOT forbid
2. The distinction between connection state (what statelessness prohibits) and application state (what the database stores)
3. Exactly how a JWT token allows stateless authentication — include a decoded payload example and explain what the server checks on each request without a database lookup
4. The scalability benefit of statelessness, using a load balancer scenario with 10 servers
5. The one legitimate reason to choose server-side session storage over JWTs — and the tradeoff it introduces

---

# Level 5 Solutions

## Solution 5.1: The Padlock

**1. The padlock is not cosmetic:**
TLS encrypts every byte of the HTTP message before it leaves the client's network stack, and decrypts it only after it arrives at the server. The padlock means: the HTTP request, including all headers and the body, was transformed into ciphertext before transmission. It is not a badge. It is proof of encryption.

**2. Wire comparison:**
```
WITHOUT TLS (plain HTTP):
──────────────────────────────────────────────────────────────
An attacker on the same network sees readable text:

POST /api/auth/login HTTP/1.1
Host: internal.mycompany.com
Content-Type: application/json

{"username": "admin", "password": "CompanySecret2025!"}

Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.abc
──────────────────────────────────────────────────────────────

WITH TLS (HTTPS):
──────────────────────────────────────────────────────────────
An attacker on the same network sees meaningless ciphertext:

17 03 03 01 2f 4a 9c f1 2e 38 d7 b5 44 02 a8 1f
c3 7e 51 90 ee d4 8b 26 f7 13 55 a2 0c 69 3d e8
[...indistinguishable from random noise...]
──────────────────────────────────────────────────────────────
```

**3. Why "private Kubernetes cluster" is insufficient:**
- **Compromised internal service:** If any pod inside the cluster is compromised (via a supply chain attack, unpatched CVE, or misconfigured container), that pod can sniff plain HTTP traffic on the cluster's internal network. All requests routed through it are visible.
- **Insider threat:** A developer, SRE, or contractor with network access can use standard tools to capture unencrypted inter-service traffic without compromising any application code. Kubernetes networking is not encrypted by default between pods.

**4. The three TLS guarantees:**

| Guarantee | Without it |
|---|---|
| **Encryption** | Passwords, tokens, and PII travel as readable text. Any network observer captures them without effort |
| **Authentication** | An attacker can impersonate your server. Clients submit credentials to the attacker's machine believing they are talking to the real server (man-in-the-middle phishing) |
| **Integrity** | An attacker can silently modify data in transit — change `amount: 1` to `amount: 10000`, flip `approved: false` to `approved: true`. The receiver has no way to detect the modification |

**5. When plain HTTP is acceptable:**
`http://localhost` — traffic on the loopback interface (`127.0.0.1`) never leaves the machine. There is no physical or virtual network path for an attacker to position themselves on. Man-in-the-middle is physically impossible. This is the only context where plain HTTP is safe. Every other communication that crosses any network boundary — including private corporate networks, VPNs, and internal Kubernetes service meshes — should use TLS.

---

## Solution 5.2: The CORS Misconception

**1. Who enforces CORS and when:**
CORS is enforced exclusively by web browsers. The enforcement occurs *after* the response has been received — the browser inspects the response headers and decides whether to expose the response body to the JavaScript code that initiated the fetch. The server sent the response. The network delivered it. The browser discarded it before the script could read it.

**2. The bypass script:**
```python
import httpx

# No browser. No Same-Origin Policy. No CORS.
response = httpx.get(
    "https://api.mycompany.com/users",
    headers={"Authorization": "Bearer a_stolen_token"}
)
print(response.json())  # Full response. CORS is irrelevant here.
```
Any attacker using curl, Python, Postman, or a custom script has full access regardless of your `allow_origins` configuration. CORS is simply not in their request path.

**3. What CORS actually protects against:**
CORS protects the *browser user* against **cross-site request forgery via JavaScript**. The threat model: a malicious website (`evil.com`) includes JavaScript that silently calls `https://api.mycompany.com/transfer-funds`. If the victim's browser has a valid session cookie for `mycompany.com`, the browser would automatically attach it to the request. Without CORS, `evil.com`'s JavaScript could read the API response and conduct the operation completely invisibly. CORS prevents this by refusing to expose cross-origin responses to scripts unless the server explicitly consents.

**4. CORS and authentication are orthogonal:**

| | Protects against | Enforced by | Bypassed by |
|---|---|---|---|
| CORS | Browser JS using victim's cookies cross-site | Browser | curl, Python, any non-browser tool |
| Authentication | Any caller without valid credentials | Server | Nothing — everyone must authenticate |

You need both. A properly CORS-configured but unauthenticated API is wide open to every automated tool in existence. A properly authenticated but CORS-unconfigured API allows `evil.com` to use your logged-in users' sessions to call your API silently.

**5. Minimum correct architecture:**
```
Layer 1 — Authentication (server-enforced):
Every request carries a valid JWT or API key.
Applied to ALL callers regardless of how the request was made.

Layer 2 — CORS (browser-enforced):
Restricts which browser origins may make cross-origin requests
and whether credentials may be sent cross-origin.
Applied only to browser-based JavaScript.

Result:
- An unauthenticated attacker script → rejected by Layer 1
- A browser on evil.com with a stolen token → rejected by Layer 2
- A browser on app.mycompany.com with a valid token → passes both
```

---

## Solution 5.3: Stateless Means No Sessions

**1. The precise definition:**
Stateless means: the server does not retain any information about individual client connections in process memory between requests. Each request must contain everything the server needs to process it. It does NOT mean: the server has no knowledge of users, no database records, no persistent data, or no concept of identity.

**2. Connection state vs application state:**
- **Connection state** (prohibited): The server keeps a Python dict `{"client_socket_123": {"user_id": 42, "last_page": "/dashboard"}}` that persists between requests. If the server restarts, this is lost. If a load balancer routes the next request to a different server, that server doesn't have the dict.
- **Application state** (fine): User records, orders, preferences, and permissions stored in PostgreSQL. They exist independently of which server processes the next request. Every server can access the same database.

Statelessness prohibits the first. It says nothing about the second.

**3. JWT stateless authentication — concrete example:**
```
Decoded JWT payload (what lives inside the token the client sends):
{
    "sub": "user_42",           ← User's ID — no DB lookup needed
    "role": "admin",            ← Permission level — already in the token
    "exp": 1739094600,          ← Expiry timestamp — server checks this
    "iss": "api.mycompany.com"  ← Who issued this token
}

The token was cryptographically SIGNED with the server's secret key.

Per-request server validation (no database query):
1. Receive: Authorization: Bearer eyJhbGci...
2. Decode the JWT
3. Verify the signature with the secret key — proves it wasn't tampered with
4. Check exp > current time — proves it hasn't expired
5. Read sub = "user_42" and role = "admin" from the payload
6. Proceed — the user is authenticated and identified

The user's identity traveled inside the request itself.
No session table, no Redis lookup, no in-memory store.
```

**4. Load balancer scalability:**
With server-side sessions: a user authenticates on Server A. Server A stores `{"session_abc": {user_id: 42}}` in memory. The load balancer must route all this user's subsequent requests to Server A (sticky sessions). When Server A is overloaded, the load balancer cannot distribute this user's traffic to Servers B, C, or D. Adding Server E does not help this user.

With JWT: the user's identity is in the token. Server A, B, C, D, and E all validate the same way — they all know the same signing secret, and they all check the same fields. The load balancer routes each request to whichever server is least loaded. You add Server E and immediately gain capacity for all users. Scaling is linear and unrestricted.

**5. When server-side sessions are appropriate:**
Server-side sessions (typically stored in Redis) are the correct choice when **immediate token revocation** is required. A JWT is valid until its expiry timestamp — you cannot "un-sign" a token. If an admin account is compromised and you invalidate its credentials, the attacker's JWT remains valid until it expires (potentially hours). With server-side sessions, you delete the Redis session entry and the next request is immediately rejected. The tradeoff: every request now requires a Redis lookup, adding latency and introducing Redis as a dependency in the critical authentication path.