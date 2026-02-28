# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "FastAPI is a smart framework. It figures out what I mean from context — `Optional` means optional, you send errors by returning them, and the order you write routes in doesn't really matter because it matches the right one. The type hints are just there to make the IDE happy."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | `Optional[str]` makes a parameter optional | Only a **default value** makes a parameter optional. `Optional[str]` only permits `None` as a type — it says nothing about whether the client must send the field |
| 2 | `return HTTPException(...)` sends an HTTP error response | `return` serialises the exception **object** as a 200 JSON blob. You must `raise` it |
| 3 | FastAPI matches routes intelligently — order doesn't matter | Routes are matched **top-to-bottom**. A dynamic `{param}` will swallow any static path defined after it |
| 4 | `response_model` just documents what the response looks like | It also **strips fields** — any key in your return value that isn't in the model is silently removed before the client receives it |
| 5 | `async def` is always better in FastAPI routes | `async def` + a blocking call **freezes the entire server**. Regular `def` offloads to a thread pool safely |
| 6 | The type annotation is metadata — FastAPI knows where data comes from by name or context | FastAPI reads annotations **at runtime** to decide: path param, query param, or request body. The type IS the routing decision |
| 7 | `status_code=204` sends `{}` or `null` if you don't return anything meaningful | Returns **literally nothing** — an empty body — even if your function explicitly returns a value |
| 8 | You can mix `Form()` fields and a Pydantic JSON body in the same endpoint | A request has one `Content-Type`. They are **mutually exclusive** |

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behaviour | Root Cause |
|---|---|---|---|
| `Optional[str]` = optional | `GET /search` returns 200 | Returns 422 — field required | Type ≠ default value |
| Route order doesn't matter | `/reports/featured` returns static data | Returns dynamic route's data (`report_id="featured"`) | Top-to-bottom matching |
| `return HTTPException(...)` sends error | `GET /items/99` returns 404 | Returns 200 with exception object as JSON | `return` never triggers the exception handler |
| `async def` is always faster | Two concurrent requests serve independently | Second request waits for first to finish | `time.sleep` in `async def` blocks the event loop |
| `response_model` is documentation | All return dict keys reach the client | Only model-declared fields survive | `response_model` is a runtime whitelist |
| `status_code=204` returns nothing only when you return `None` | `return {"deleted": item}` sends the item back | Client receives empty body regardless | HTTP 204 = no content, FastAPI discards the return value |

---

## The Main Purpose (Essence)

**Core Insight:** FastAPI is a **type-hint-driven contract system**. Routing, validation, serialisation, documentation, injection — every feature derives from reading Python type annotations at runtime. Understanding this one principle decodes every behaviour that looks like "magic."

**The Problems Being Solved:**

| Problem | Without FastAPI | With FastAPI |
|---|---|---|
| Routing | `if method == "GET" and path == "/..."` | `@app.get("/...")` |
| Type conversion | `int(path.split("/")[-1])` + `try/except` | `param: int` in signature |
| Validation | Manual `if` checks on every field | Pydantic model as function parameter |
| Error responses | Build status + JSON manually | `raise HTTPException(status_code=..., detail=...)` |
| Filtering sensitive fields | Manually exclude from every return | `response_model=OutputModel` |
| Documentation | Write and maintain separately | Auto-generated from type hints |

**The Key Distinctions:**
- `Optional[str]` = "this value can be `None`." A **default value** (`= None`) = "the client doesn't have to send this."
- `raise HTTPException(...)` = Python's exception mechanism, intercepted by FastAPI's error handler, becomes a real HTTP error. `return HTTPException(...)` = just a Python object serialised as 200 JSON.
- `async def` + `await` = cooperative, non-blocking. `async def` + `time.sleep()` = a broken promise to the event loop that freezes everything.

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Optional That Isn't

```python
from fastapi import FastAPI
from typing import Optional

app = FastAPI()

@app.get("/search")
async def search_products(
    query: Optional[str],              # A
    category: str = "all",             # B
    max_results: Optional[int] = 10,   # C
    in_stock: bool = True,             # D
):
    return {
        "query": query,
        "category": category,
        "max_results": max_results,
        "in_stock": in_stock,
    }
```

**Questions:**

1. **Predict.** For each parameter, fill in the table without running the code:

| Parameter | Required or Optional? | Default value (if optional) |
|---|---|---|
| A: `query: Optional[str]` | | |
| B: `category: str = "all"` | | |
| C: `max_results: Optional[int] = 10` | | |
| D: `in_stock: bool = True` | | |

2. **Predict the HTTP response** (status code + response body) for each request:
   - `GET /search`
   - `GET /search?query=python`
   - `GET /search?category=books&max_results=5`

3. **State the rule.** In one precise sentence: what is the *only* thing that makes a FastAPI query parameter optional?

---

## Exercise 1.2: The Route That Ate Its Neighbour

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/reports/{report_id}")
async def get_report(report_id: str):
    return {"source": "dynamic", "report_id": report_id}

@app.get("/reports/latest")
async def get_latest():
    return {"source": "static", "name": "latest"}

@app.get("/reports/summary")
async def get_summary():
    return {"source": "static", "name": "summary"}
```

**Questions:**

1. **Predict the response body** for each request:
   - `GET /reports/latest`
   - `GET /reports/summary`
   - `GET /reports/42`

2. **Explain the mechanism.** For `GET /reports/latest`, trace exactly why the response comes from `get_report` and not `get_latest`. What specific step in FastAPI's matching logic causes this?

3. **Mental trace.** Walk through FastAPI's route-matching process for `GET /reports/latest` step by step.

4. **Fix it.** Reorder the routes so that `GET /reports/latest` returns `{"source": "static", "name": "latest"}`. State the rule you are applying and connect it to a pattern you already know from Python.

---

## Exercise 1.3: The Error That Always Returns 200

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

items = {1: "Laptop", 2: "Mouse", 3: "Keyboard"}

@app.get("/items/{item_id}")
async def get_item(item_id: int):
    if item_id not in items:
        return HTTPException(status_code=404, detail="Item not found")
    return {"id": item_id, "name": items[item_id]}

@app.delete("/items/{item_id}")
async def delete_item(item_id: int):
    if item_id not in items:
        return HTTPException(status_code=404, detail="Item not found")
    del items[item_id]
    return HTTPException(status_code=204, detail="Deleted")
```

**Questions:**

1. **Predict** the HTTP status code and response body for each:

| Request | Status Code | Response Body |
|---|---|---|
| `GET /items/99` (missing) | | |
| `GET /items/1` (exists) | | |
| `DELETE /items/99` (missing) | | |
| `DELETE /items/1` (exists) | | |

2. **Explain the first bug.** Why does `return HTTPException(status_code=404, ...)` not produce a 404 response? Trace what FastAPI actually does with that return value.

3. **Explain the second bug.** In `delete_item`, the developer tried to signal a 204 by returning `HTTPException(status_code=204, ...)`. Even if they fixed the `return` → `raise` problem, why would `raise HTTPException(status_code=204, detail="Deleted")` still be semantically wrong?

4. **Fix both routes.** Write corrected versions. `delete_item` should return 204 with no body on success.

---

## Exercise 1.4: The Server That Froze

```python
import time
from fastapi import FastAPI

app = FastAPI()

@app.get("/report")
async def generate_report():
    """Runs a heavy operation. Takes ~3 seconds."""
    time.sleep(3)
    return {"report": "complete"}

@app.get("/ping")
async def ping():
    """Should always respond instantly."""
    return {"status": "alive"}
```

Two HTTP requests arrive at the server at exactly the same millisecond:
- **Request A:** `GET /report` (arrives at T=0ms)
- **Request B:** `GET /ping` (arrives at T=0ms)

**Questions:**

1. **Predict.** At approximately what time does Request B receive its response?
   - A) T≈0ms (immediately, the report runs in the background)
   - B) T≈3000ms (after the report finishes)
   - C) T≈1500ms (halfway through, round-robin scheduling)
   - D) It errors with a timeout

2. **Explain why.** Name the exact thing `time.sleep` is blocking. Why does blocking that thing affect `GET /ping` at all?

3. **Compare.** If `generate_report` were changed from `async def` to regular `def`, would Request B still be delayed? Trace through what FastAPI does differently with a synchronous route function.

4. **The fix.** If the 3-second wait were `await asyncio.sleep(3)` instead of `time.sleep(3)`, what would happen to Request B? Why is the outcome completely different?

---

# Level 1 Solutions

## Solution 1.1

**1. Correct table:**

| Parameter | Required or Optional? | Default value |
|---|---|---|
| A: `query: Optional[str]` | **Required** | — (no default) |
| B: `category: str = "all"` | Optional | `"all"` |
| C: `max_results: Optional[int] = 10` | Optional | `10` |
| D: `in_stock: bool = True` | Optional | `True` |

Parameter A is the trap. `Optional[str]` is a **type declaration** — it says the value may be `None`. It makes no claim about whether the client must provide it. There is no `= something` on the right, so FastAPI marks it required.

**2. Requests:**
- `GET /search` → **422 Unprocessable Entity.** `query` has no default — FastAPI demands it from the client and rejects the request.
- `GET /search?query=python` → **200.** `{"query": "python", "category": "all", "max_results": 10, "in_stock": true}`
- `GET /search?category=books&max_results=5` → **422.** `query` is still absent. Providing other parameters doesn't satisfy a required one.

**3. The rule:**
A FastAPI query parameter is optional if and only if it has an explicit default value (`= None`, `= 0`, `= "all"`, etc.). The type annotation is irrelevant to optionality.

**The Deeper Insight:**
```
Optional[str]            → type allows None, still REQUIRED (no = sign)
Optional[str] = None     → type allows None, OPTIONAL (default is None)
str = "all"              → type requires str, OPTIONAL (default is "all")
```
`Optional` and "optional" sound the same but mean different things. `Optional[str]` is Python typing. "optional parameter" is a FastAPI routing concept. The `=` separates them.

---

## Solution 1.2

**1. Predictions:**
- `GET /reports/latest` → **200,** `{"source": "dynamic", "report_id": "latest"}`
- `GET /reports/summary` → **200,** `{"source": "dynamic", "report_id": "summary"}`
- `GET /reports/42` → **200,** `{"source": "dynamic", "report_id": "42"}`

`get_latest` and `get_summary` are **never called** in this configuration.

**2. Mechanism:**
FastAPI stores routes in an ordered list. When `GET /reports/latest` arrives, it checks route 1: `/reports/{report_id}`. The `{report_id}` is typed as `str`. The string `"latest"` successfully matches the wildcard and successfully converts to `str`. FastAPI calls `get_report("latest")`. It stops. Routes 2 and 3 are never checked.

**3. Mental trace:**
```
Incoming: GET /reports/latest

Route 1: /reports/{report_id}  (report_id: str)
  Pattern match: "latest" fits {report_id}? YES
  Type conversion: str("latest") → "latest" ✓
  MATCH. Call get_report(report_id="latest").
  Halt. Do not check route 2 or 3.
```

**4. Fix — static routes before dynamic routes:**
```python
@app.get("/reports/latest")     # Static first — no wildcard to steal it
async def get_latest():
    return {"source": "static", "name": "latest"}

@app.get("/reports/summary")    # Static first
async def get_summary():
    return {"source": "static", "name": "summary"}

@app.get("/reports/{report_id}") # Dynamic last — catches everything else
async def get_report(report_id: str):
    return {"source": "dynamic", "report_id": report_id}
```

**The rule:** Specific paths before generic paths. This is the same principle as exception handling: `except SpecificError:` must come before `except Exception:`. A catch-all placed first consumes everything; it must be last.

---

## Solution 1.3

**1. Predictions:**

| Request | Status Code | Response Body |
|---|---|---|
| `GET /items/99` | **200 OK** | `{"status_code": 404, "detail": "Item not found"}` |
| `GET /items/1` | 200 OK | `{"id": 1, "name": "Laptop"}` |
| `DELETE /items/99` | **200 OK** | `{"status_code": 404, "detail": "Item not found"}` |
| `DELETE /items/1` | **200 OK** | `{"status_code": 204, "detail": "Deleted"}` |

**2. The first bug — `return` vs `raise`:**
`HTTPException(status_code=404, detail="...")` is Python object construction — it creates an instance of the `HTTPException` class with those attributes. `return` hands this object to FastAPI's serialiser. FastAPI serialises it to JSON like any other return value. The HTTP status line stays 200 because nothing ever told the HTTP layer to change it. The `status_code=404` ends up as a JSON field inside a 200 response — data, not a protocol signal.

`raise` is different. It activates Python's exception propagation system. FastAPI registers an exception handler specifically for `HTTPException`. When one is raised, that handler intercepts it, reads the `status_code`, and constructs a real HTTP error response with that status code. The client receives an actual 404.

```
return HTTPException(...) → FastAPI serialiser → {"status_code": 404, ...} → 200 response
raise HTTPException(...)  → Python exception → FastAPI exception handler → real 404 response
```

**3. The second bug — `HTTPException` for success codes:**
HTTP 204 "No Content" means the server succeeded and has nothing to send back. Its defining characteristic is an empty body. `HTTPException` always sends a JSON body: `{"detail": "..."}`. These are contradictory. Even with `raise`, you would send a 204 response with a JSON body, which violates the HTTP spec. `HTTPException` belongs in the 4xx and 5xx family — error conditions. Success codes are declared on the decorator with `status_code=`.

**4. Fixed code:**
```python
@app.get("/items/{item_id}")
async def get_item(item_id: int):
    if item_id not in items:
        raise HTTPException(status_code=404, detail="Item not found")  # raise
    return {"id": item_id, "name": items[item_id]}

@app.delete("/items/{item_id}", status_code=204)  # 204 on the decorator
async def delete_item(item_id: int):
    if item_id not in items:
        raise HTTPException(status_code=404, detail="Item not found")  # raise
    del items[item_id]
    return None  # FastAPI ignores this; body will be empty
```

---

## Solution 1.4

**1. Answer: B) T≈3000ms**

Request B does not respond until `generate_report` finishes.

**2. What is being blocked:**
FastAPI runs on a single-threaded event loop — one thread. `async def` runs coroutines directly on that thread. When `time.sleep(3)` is called, it is a blocking OS call. It freezes the OS thread entirely. That thread IS the event loop. While the thread is frozen, no other coroutine can run — including the `ping` handler. The entire server is paralysed for 3 seconds.

```
T=0ms:      Event loop starts generate_report (async def)
T=0ms:      time.sleep(3) called → OS thread frozen
T=0–3000ms: Event loop is frozen. ping handler cannot run.
T=3000ms:   time.sleep returns → event loop resumes
T=3000ms:   ping can now run → responds to Request B
```

**3. Regular `def` — no delay for Request B:**
FastAPI detects synchronous (`def`) route functions and dispatches them to a **thread pool** via `asyncio.run_in_executor`. The blocking work runs on a separate OS thread. The event loop thread itself is never blocked. Request B's `ping` coroutine runs on the event loop immediately at T≈0ms.

```
T=0ms:   generate_report (def) → dispatched to thread pool thread
T=0ms:   Event loop thread is FREE
T=0ms:   ping coroutine runs on event loop → Request B responds immediately
T=3000ms: thread pool thread finishes → Request A responds
```

**4. `await asyncio.sleep(3)` — Request B responds at T≈0ms:**
`asyncio.sleep` is an async-aware wait. When the coroutine hits `await asyncio.sleep(3)`, it **yields control** back to the event loop. The event loop is now free to pick up the next waiting work — the `ping` handler. Request B responds immediately. Three seconds later, the event loop returns to `generate_report` and finishes it.

**The Key Insight:**
`async def` is not a performance guarantee. It is a *promise* to the event loop: "When I wait, I will yield so you can do other things." `time.sleep` breaks that promise silently. The event loop trusts coroutines to cooperate. The consequence of breaking that trust is worse than using `def` — at least `def` is honest about its blocking and FastAPI routes it to a thread pool.

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The API That Leaks Secrets

**Context:** A developer built a user registration endpoint. It runs without errors. Users are created successfully. But a security engineer just flagged a critical issue in the code review — something in the response body that should never reach a client.

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class UserCreate(BaseModel):
    username: str
    email: str
    password: str

def simulate_db_insert(user: UserCreate) -> dict:
    """Simulates saving to DB and returning the full database row."""
    return {
        "id": 1001,
        "username": user.username,
        "email": user.email,
        "hashed_password": f"$2b$12$FakeHashOf{user.password}",
        "is_admin": False,
        "internal_notes": "Registered via public API",
        "cost_center": "API-EXTERNAL",
    }

@app.post("/users", status_code=201)
async def register_user(user: UserCreate):
    db_row = simulate_db_insert(user)
    return db_row
```

---

**Task 2a: Identify the Flaw**

1. What does the full response body look like when you `POST /users` with `{"username": "alice", "email": "alice@test.com", "password": "hunter2"}`?
2. Which specific fields in that response should never reach a client?
3. This code raises no errors or warnings. Why does FastAPI not protect against this automatically?

---

**Task 2b: Explain the Mechanism**

1. What does FastAPI do with the value you `return` from a route function when no `response_model` is set?
2. Does `status_code=201` protect the response content in any way? Why or why not?
3. Describe in one sentence what `response_model` does at runtime — not what it documents.

---

**Task 2c: Instrument to Prove the Diagnosis**

Without fixing the security flaw, add one line that prints the full list of keys being sent before the response is dispatched. Show the terminal output that confirms the leakage.

```python
@app.post("/users", status_code=201)
async def register_user(user: UserCreate):
    db_row = simulate_db_insert(user)
    # ADD ONE PRINT HERE
    return db_row
```

---

**Task 2d: Fix the Code**

1. Define a `UserResponse` Pydantic model that exposes only: `id`, `username`, `email`.
2. Add `response_model=UserResponse` to the decorator.
3. Do **not** change `simulate_db_insert` and do **not** change the `return db_row` line. The filtering must happen automatically via `response_model`.

---

**Task 2e: Write a Verification Test**

Write a test using `TestClient` that **fails** on the original code and **passes** on the fixed code:

```python
from fastapi.testclient import TestClient
# client = TestClient(app)

def test_sensitive_fields_never_exposed():
    response = client.post("/users", json={
        "username": "alice",
        "email": "alice@test.com",
        "password": "hunter2"
    })
    assert response.status_code == 201
    data = response.json()

    # Write assertions that fail on the original, pass on the fix:
    assert ???
    assert ???
    # Write assertions that must pass on both:
    assert ???
    assert ???
```

---

## Exercise 2.2: The 204 With Something to Say

**Context:** The product team wants a delete endpoint that returns the deleted resource so the frontend can show a confirmation ("You deleted: Laptop × 2") and offer an undo option. The developer wrote the following code and shows you their Postman screenshot: "Look, no errors! It returns 204." But the frontend team is filing bugs every day: "The response body is always empty."

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

orders: dict[int, dict] = {
    1: {"id": 1, "product": "Laptop", "quantity": 2},
    2: {"id": 2, "product": "Mouse", "quantity": 5},
}

@app.delete("/orders/{order_id}", status_code=204)
async def delete_order(order_id: int):
    if order_id not in orders:
        raise HTTPException(status_code=404, detail="Order not found")

    deleted = orders.pop(order_id)
    return {
        "message": "Order deleted",
        "deleted_order": deleted,
    }
```

---

**Task 2a: Identify the Flaw**

1. What does `DELETE /orders/1` actually return to the client?
2. The return dict IS computed inside the function — the code runs `orders.pop(order_id)` and builds the dict. Why doesn't the client see it?
3. Is the `404` case working correctly? Explain why the `raise` path behaves differently from the `return` path.

---

**Task 2b: Explain the Mechanism**

1. What does HTTP 204 "No Content" mean at the protocol level?
2. When FastAPI sees `status_code=204` on the decorator, what does it do with your return value?
3. Why is this FastAPI behaviour *correct* per the HTTP spec, even though it frustrates this particular developer?

---

**Task 2c: Instrument to Prove the Diagnosis**

Add a `print` statement inside the function to prove that the return value is computed but discarded. Show both the terminal output and the HTTP response side by side.

```python
@app.delete("/orders/{order_id}", status_code=204)
async def delete_order(order_id: int):
    if order_id not in orders:
        raise HTTPException(status_code=404, detail="Order not found")
    deleted = orders.pop(order_id)
    payload = {"message": "Order deleted", "deleted_order": deleted}
    # ADD PRINT HERE
    return payload
```

---

**Task 2d: Fix the Code**

The developer's intent is legitimate — returning the deleted resource is a valid API design. Fix the endpoint so that:
1. A successful delete returns the deleted order AND the semantically correct status code.
2. Not-found still returns 404.

You must choose a status code that permits a response body. Explain your choice.

---

**Task 2e: Write a Verification Test**

```python
from fastapi.testclient import TestClient
# client = TestClient(app)

def test_delete_returns_deleted_order():
    response = client.delete("/orders/1")
    # Write assertions that:
    # 1. Confirm the status code is NOT 204
    # 2. Confirm the response body contains the deleted order
    assert ???
    assert ???

def test_delete_nonexistent_returns_404():
    response = client.delete("/orders/999")
    assert ???
    assert ???
```

---

# Level 2 Solutions

## Solution 2.1: The API That Leaks Secrets

**Task 2a: The Flaw**

1. Full response body:
```json
{
    "id": 1001,
    "username": "alice",
    "email": "alice@test.com",
    "hashed_password": "$2b$12$FakeHashOfhunter2",
    "is_admin": false,
    "internal_notes": "Registered via public API",
    "cost_center": "API-EXTERNAL"
}
```

2. Must never reach a client: `hashed_password`, `is_admin`, `internal_notes`, `cost_center`.

3. FastAPI has no way to know your business rules without `response_model`. Its default behaviour is: "serialise whatever the function returns and send it." A dict is valid. FastAPI trusts you to return only what is appropriate. Without a declared output contract, it sends everything.

**Task 2b: The Mechanism**

1. Without `response_model`, FastAPI passes the return value to `jsonable_encoder`, wraps it in a `JSONResponse`, and sends it. No filtering. No inspection.

2. `status_code=201` controls only the HTTP status line. `201 Created` and `201 Created with leaked passwords` are equally valid uses of `status_code=201`. The status code says nothing about body content.

3. `response_model` is a runtime filter: it validates your return value against the Pydantic model and removes every key not declared in that model before serialisation.

**Task 2c: Instrumentation**

```python
@app.post("/users", status_code=201)
async def register_user(user: UserCreate):
    db_row = simulate_db_insert(user)
    print(f"DEBUG — sending keys: {list(db_row.keys())}")
    return db_row

# Terminal output:
# DEBUG — sending keys: ['id', 'username', 'email', 'hashed_password', 'is_admin', 'internal_notes', 'cost_center']
```

**Task 2d: Fix**

```python
class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    # hashed_password: absent — intentional
    # is_admin: absent — intentional
    # internal_notes: absent — intentional
    # cost_center: absent — intentional

@app.post("/users", status_code=201, response_model=UserResponse)
async def register_user(user: UserCreate):
    db_row = simulate_db_insert(user)
    return db_row  # Unchanged — response_model strips it automatically
```

**Task 2e: Verification Test**

```python
from fastapi.testclient import TestClient

client = TestClient(app)

def test_sensitive_fields_never_exposed():
    response = client.post("/users", json={
        "username": "alice",
        "email": "alice@test.com",
        "password": "hunter2"
    })
    assert response.status_code == 201
    data = response.json()

    # These FAIL on the original, PASS on the fixed version:
    assert "hashed_password" not in data, "Password hash was leaked!"
    assert "is_admin" not in data, "Admin flag was leaked!"
    assert "internal_notes" not in data, "Internal notes were leaked!"
    assert "cost_center" not in data, "Cost centre was leaked!"

    # These must PASS on both:
    assert data["username"] == "alice"
    assert data["email"] == "alice@test.com"
    assert "id" in data
```

**Why the Naive Model Fails:**
The naive model assumes that if you "don't put the bad data in the return statement," it won't be sent. But in a real codebase, `simulate_db_insert` might be a SQLAlchemy ORM object with 20 columns, maintained by someone else. Relying on every developer to manually exclude sensitive fields from every `return` statement is a fragile social contract. `response_model` is a static, enforceable whitelist declared at the API boundary. A future developer adding `hashed_password` to the return dict cannot override it.

---

## Solution 2.2: The 204 With Something to Say

**Task 2a: The Flaw**

1. `DELETE /orders/1` returns: an **empty body**. Literally nothing — not `{}`, not `null`.
2. The dict IS computed — `orders.pop(order_id)` runs, the dict is built. But `status_code=204` on the decorator signals FastAPI: "this response has no content." FastAPI wraps the response in a `Response` class (not `JSONResponse`) with no body. Your return value is discarded before serialisation.
3. The `404` case works correctly. `raise HTTPException(...)` is an exception — it bypasses the normal response-building path entirely. FastAPI's exception handler for `HTTPException` runs instead. That handler is not subject to the `status_code=204` rule on the decorator. It sends its own response with its own body.

**Task 2b: The Mechanism**

1. HTTP 204 is a protocol-level declaration: "The request succeeded and there is nothing to communicate back." HTTP clients that respect the spec must not render a body, even if one is accidentally sent. The client sees an empty response by design.

2. FastAPI sees `status_code=204` and returns `Response(content=None, status_code=204)`. The serialisation step that would normally convert your return value into JSON is skipped entirely.

3. This is correct behaviour. When you declare `status_code=204`, you are promising the client: "I have nothing to show you." If you have something to show, that is a different contract — use a status code that permits a body.

**Task 2c: Instrumentation**

```python
@app.delete("/orders/{order_id}", status_code=204)
async def delete_order(order_id: int):
    if order_id not in orders:
        raise HTTPException(status_code=404, detail="Order not found")
    deleted = orders.pop(order_id)
    payload = {"message": "Order deleted", "deleted_order": deleted}
    print(f"DEBUG — I computed: {payload}")   # This runs
    return payload  # FastAPI discards this

# Terminal:
# DEBUG — I computed: {'message': 'Order deleted', 'deleted_order': {...}}
#
# HTTP response:
# HTTP/1.1 204 No Content
# (body is empty)
```

**Task 2d: Fix**

Use `status_code=200`. The developer wants to return the deleted resource — that requires a body:

```python
from pydantic import BaseModel

class OrderInDB(BaseModel):
    id: int
    product: str
    quantity: int

class DeleteResponse(BaseModel):
    message: str
    deleted_order: OrderInDB

@app.delete("/orders/{order_id}", status_code=200, response_model=DeleteResponse)
async def delete_order(order_id: int):
    if order_id not in orders:
        raise HTTPException(status_code=404, detail="Order not found")
    deleted = orders.pop(order_id)
    return {"message": "Order deleted", "deleted_order": deleted}
```

**Status code decision:** Use 204 when delete is "silent" — the resource is gone, nothing to confirm. Use 200 when you return the deleted resource, a confirmation summary, or an undo token. Pick one convention and enforce it across your entire API.

**Task 2e: Verification Test**

```python
from fastapi.testclient import TestClient

client = TestClient(app)

def test_delete_returns_deleted_order():
    response = client.delete("/orders/1")
    assert response.status_code == 200          # NOT 204
    data = response.json()
    assert "deleted_order" in data
    assert data["deleted_order"]["product"] == "Laptop"
    assert data["deleted_order"]["quantity"] == 2

def test_delete_nonexistent_returns_404():
    response = client.delete("/orders/999")
    assert response.status_code == 404
    assert "not found" in response.json()["detail"].lower()
```

---

# LEVEL 2.5: EXTEND EXISTING CODE (THE INTEGRATION)

## Exercise 2.5: The Endpoint That Grew Up

**Background:** A teammate wrote a basic Product CRUD API that passes all existing tests. You have been asked to harden it in three iterations before it ships to production.

**Existing code:**

```python
# main.py
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class ProductCreate(BaseModel):
    name: str
    price: float
    cost_price: float    # Internal margin field — must NEVER appear in responses
    stock: int = 0

products_db: dict[int, dict] = {}
_next_id = 1

@app.get("/products/{product_id}")
async def get_product(product_id: int):
    return products_db.get(product_id)   # Returns None if missing — no 404

@app.get("/products")
async def list_products(limit: Optional[int], skip: int = 0):
    items = list(products_db.values())
    return items[skip : skip + limit]

@app.post("/products")
async def create_product(product: ProductCreate):
    global _next_id
    new_product = {
        "id": _next_id,
        "name": product.name,
        "price": product.price,
        "cost_price": product.cost_price,   # Leaking internal data
        "stock": product.stock,
    }
    products_db[_next_id] = new_product
    _next_id += 1
    return new_product

@app.put("/products/{product_id}")
async def update_product(product_id: int, product: ProductCreate):
    products_db[product_id] = {"id": product_id, **product.model_dump()}
    return products_db[product_id]
```

**Existing tests — these must pass after every extension:**

```python
# test_products.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_create_product():
    response = client.post("/products", json={
        "name": "Widget", "price": 9.99, "cost_price": 3.50, "stock": 100
    })
    assert response.status_code in (200, 201)
    assert response.json()["name"] == "Widget"

def test_get_product():
    # Assumes at least one product has been created (run after test_create_product)
    response = client.get("/products/1")
    assert response.status_code == 200

def test_list_products():
    response = client.get("/products?limit=10")
    assert response.status_code == 200
    assert isinstance(response.json(), list)
```

---

### Extension 1: Response Model and Correct Status Codes

**Task:**

1. Define a `ProductResponse` Pydantic model with fields: `id`, `name`, `price`, `stock`. `cost_price` is deliberately absent.
2. Add `response_model=ProductResponse` to `get_product`, `create_product`, and `update_product`.
3. Fix `create_product` to return status code **201**.
4. Fix the latent bug in `list_products`: `limit: Optional[int]` without a default is required, which means `GET /products` without `?limit=` would 422. Fix it with a sensible default.

**Constraint:** Do NOT change what the function bodies return. All filtering must happen through `response_model`.

**New tests that must pass after Extension 1:**

```python
def test_cost_price_never_in_response():
    response = client.post("/products", json={
        "name": "Secret", "price": 5.0, "cost_price": 1.0, "stock": 10
    })
    assert "cost_price" not in response.json()

def test_create_returns_201():
    response = client.post("/products", json={
        "name": "Gadget", "price": 19.99, "cost_price": 7.00, "stock": 50
    })
    assert response.status_code == 201
```

---

### Extension 2: Proper Error Handling

**Task:** Replace silent failures with explicit HTTP errors.

1. `get_product` currently returns `None` when a product isn't found — FastAPI sends `null` with a 200. Fix: raise 404.
2. `update_product` currently creates a product if the ID doesn't exist (silent upsert). Fix: raise 404 if the product does not exist.
3. `list_products`: if `skip` is negative, raise a 400 with detail `"skip must be non-negative"`.

**New tests that must pass after Extension 2 (all Extension 1 tests must still pass):**

```python
def test_get_nonexistent_returns_404():
    response = client.get("/products/9999")
    assert response.status_code == 404

def test_update_nonexistent_returns_404():
    response = client.put("/products/9999", json={
        "name": "Ghost", "price": 1.0, "cost_price": 0.5, "stock": 0
    })
    assert response.status_code == 404

def test_negative_skip_returns_400():
    response = client.get("/products?skip=-1&limit=10")
    assert response.status_code == 400
    assert "skip" in response.json()["detail"].lower()
```

---

### Extension 3: APIRouter Refactor

**Task:** Extract the product routes into their own module.

1. Create `routers/products.py`. Move all four route functions into it. Use `APIRouter(prefix="/products", tags=["Products"])`.
2. Update route paths: since the prefix is now `/products`, change `@router.get("/products/{id}")` to `@router.get("/{id}")`, etc.
3. Create `schemas/product.py`. Move `ProductCreate` and `ProductResponse` into it. Import them in the router file.
4. Update `main.py` to only contain app creation and `include_router`.

**Constraint:** The test file must not change. All tests must still pass with zero modification.

**Required final structure:**
```
project/
├── main.py
├── routers/
│   ├── __init__.py
│   └── products.py
└── schemas/
    ├── __init__.py
    └── product.py
```

---

# Level 2.5 Solutions

## Solution 2.5: Extension 1 — Response Model and Status Codes

```python
# main.py
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class ProductCreate(BaseModel):
    name: str
    price: float
    cost_price: float
    stock: int = 0

class ProductResponse(BaseModel):     # NEW
    id: int
    name: str
    price: float
    stock: int
    # cost_price: intentionally absent

products_db: dict[int, dict] = {}
_next_id = 1

@app.get("/products/{product_id}", response_model=ProductResponse)
async def get_product(product_id: int):
    return products_db.get(product_id)

@app.get("/products", response_model=list[ProductResponse])
async def list_products(limit: int = 20, skip: int = 0):   # Fixed: default for limit
    items = list(products_db.values())
    return items[skip : skip + limit]

@app.post("/products", status_code=201, response_model=ProductResponse)  # Fixed: 201
async def create_product(product: ProductCreate):
    global _next_id
    new_product = {
        "id": _next_id,
        "name": product.name,
        "price": product.price,
        "cost_price": product.cost_price,  # Still here — response_model strips it
        "stock": product.stock,
    }
    products_db[_next_id] = new_product
    _next_id += 1
    return new_product

@app.put("/products/{product_id}", response_model=ProductResponse)
async def update_product(product_id: int, product: ProductCreate):
    products_db[product_id] = {"id": product_id, **product.model_dump()}
    return products_db[product_id]
```

**Why the `limit` fix matters:** `limit: Optional[int]` with no default is a required parameter. The existing test `GET /products?limit=10` passes a limit, so it accidentally worked. But `GET /products` without any limit would 422. Changing to `limit: int = 20` gives a sensible default, makes the parameter truly optional, and the existing test still passes because it overrides the default with `?limit=10`.

---

## Solution 2.5: Extension 2 — Error Handling

```python
from fastapi import FastAPI, HTTPException
# ... (models same as Extension 1)

@app.get("/products/{product_id}", response_model=ProductResponse)
async def get_product(product_id: int):
    product = products_db.get(product_id)
    if product is None:                                       # NEW
        raise HTTPException(status_code=404, detail="Product not found")
    return product

@app.get("/products", response_model=list[ProductResponse])
async def list_products(limit: int = 20, skip: int = 0):
    if skip < 0:                                              # NEW
        raise HTTPException(status_code=400, detail="skip must be non-negative")
    items = list(products_db.values())
    return items[skip : skip + limit]

@app.post("/products", status_code=201, response_model=ProductResponse)
async def create_product(product: ProductCreate):
    global _next_id
    new_product = {
        "id": _next_id, "name": product.name, "price": product.price,
        "cost_price": product.cost_price, "stock": product.stock,
    }
    products_db[_next_id] = new_product
    _next_id += 1
    return new_product

@app.put("/products/{product_id}", response_model=ProductResponse)
async def update_product(product_id: int, product: ProductCreate):
    if product_id not in products_db:                        # NEW
        raise HTTPException(status_code=404, detail="Product not found")
    products_db[product_id] = {"id": product_id, **product.model_dump()}
    return products_db[product_id]
```

---

## Solution 2.5: Extension 3 — APIRouter Refactor

```python
# schemas/product.py
from pydantic import BaseModel

class ProductCreate(BaseModel):
    name: str
    price: float
    cost_price: float
    stock: int = 0

class ProductResponse(BaseModel):
    id: int
    name: str
    price: float
    stock: int
```

```python
# routers/products.py
from fastapi import APIRouter, HTTPException
from schemas.product import ProductCreate, ProductResponse

router = APIRouter(prefix="/products", tags=["Products"])

products_db: dict[int, dict] = {}
_next_id = 1

@router.get("/{product_id}", response_model=ProductResponse)
async def get_product(product_id: int):
    product = products_db.get(product_id)
    if product is None:
        raise HTTPException(status_code=404, detail="Product not found")
    return product

@router.get("/", response_model=list[ProductResponse])
async def list_products(limit: int = 20, skip: int = 0):
    if skip < 0:
        raise HTTPException(status_code=400, detail="skip must be non-negative")
    return list(products_db.values())[skip : skip + limit]

@router.post("/", status_code=201, response_model=ProductResponse)
async def create_product(product: ProductCreate):
    global _next_id
    new_product = {
        "id": _next_id, "name": product.name, "price": product.price,
        "cost_price": product.cost_price, "stock": product.stock,
    }
    products_db[_next_id] = new_product
    _next_id += 1
    return new_product

@router.put("/{product_id}", response_model=ProductResponse)
async def update_product(product_id: int, product: ProductCreate):
    if product_id not in products_db:
        raise HTTPException(status_code=404, detail="Product not found")
    products_db[product_id] = {"id": product_id, **product.model_dump()}
    return products_db[product_id]
```

```python
# main.py — thin by design
from fastapi import FastAPI
from routers import products

app = FastAPI()
app.include_router(products.router)
```

**Why the tests still pass without modification:** `TestClient(app)` is built from the `app` in `main.py`. `app` includes the router. The router's `prefix="/products"` combined with the route paths (`/`, `/{id}`) assembles to the exact same URLs the tests use: `/products`, `/products/1`, etc. The URL contract is unchanged. The test file is unaware of the internal refactor.

**Route order caveat in the router:** `GET /{product_id}` and `GET /` do not conflict because FastAPI distinguishes zero-segment paths (`/products/`) from one-segment paths (`/products/1`). If you later add `GET /products/featured`, that static route must be registered before `GET /{product_id}` — the same rule applies inside a router.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## Exercise 3: The Warehouse Inventory System

**Scenario:** You are building inventory management for a logistics company. The business rules below are non-negotiable, and the tests are fixed — you must write code to satisfy them.

**Business Rules:**
- Items belong to warehouses. A warehouse must exist before items can be added.
- The `supplier_code` field identifies the vendor. It must be accepted as input but **must never appear in any API response**.
- No two items in the same warehouse may share the same `name` (case-insensitive match).

**Technical Constraints:**
1. All item routes must use `APIRouter` with `prefix="/warehouses"` and `tags=["Inventory"]`.
2. `main.py` must contain only app creation and router registration.
3. Pydantic models must live in a `schemas/` directory.
4. `supplier_code` must be filtered by `response_model`, not by manually excluding it from the return dict.
5. Negative `quantity` must be rejected by Pydantic (`Field` constraint), producing a 422 automatically — no `if quantity < 0: raise HTTPException` allowed.

**Starting data (use exactly):**
```python
warehouses_db: dict[int, dict] = {
    1: {"id": 1, "name": "North Warehouse"},
    2: {"id": 2, "name": "South Warehouse"},
}
items_db: dict[int, dict] = {}
_next_item_id = 1
```

---

### Task 3a: Implement the Create Endpoint

Implement `POST /warehouses/{warehouse_id}/items`. The following tests must pass:

```python
# test_inventory.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_create_item_success():
    response = client.post("/warehouses/1/items", json={
        "name": "Bolt M8",
        "quantity": 500,
        "supplier_code": "BOLT-CORP-042"
    })
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Bolt M8"
    assert data["quantity"] == 500
    assert data["warehouse_id"] == 1
    assert "supplier_code" not in data

def test_warehouse_not_found():
    response = client.post("/warehouses/99/items", json={
        "name": "Widget", "quantity": 10, "supplier_code": "W-001"
    })
    assert response.status_code == 404

def test_duplicate_name_rejected():
    client.post("/warehouses/1/items", json={
        "name": "Nut M6", "quantity": 200, "supplier_code": "NUT-001"
    })
    response = client.post("/warehouses/1/items", json={
        "name": "nut m6",   # Same name, different case — must be caught
        "quantity": 100,
        "supplier_code": "NUT-002"
    })
    assert response.status_code == 409
```

---

### Task 3b: Add the Quantity Constraint

Add a Pydantic field constraint so that `quantity < 0` returns 422 automatically. No `if` check in the route body is permitted for this constraint.

```python
def test_negative_quantity_rejected():
    response = client.post("/warehouses/1/items", json={
        "name": "Broken Widget",
        "quantity": -5,
        "supplier_code": "BW-001"
    })
    assert response.status_code == 422
    errors = response.json()["detail"]
    # Error must reference the "quantity" field
    assert any("quantity" in str(e.get("loc", "")) for e in errors)
```

---

### Task 3c: Add the List Endpoint

Add `GET /warehouses/{warehouse_id}/items` with:
- 404 if the warehouse doesn't exist.
- Returns all items for that warehouse.
- Optional query parameter `in_stock_only: bool = False` — if `True`, return only items where `quantity > 0`.
- Uses the same `ItemResponse` model (no `supplier_code`).

All previous tests must still pass after adding this endpoint.

```python
def test_list_items_for_warehouse():
    response = client.get("/warehouses/1/items")
    assert response.status_code == 200
    assert isinstance(response.json(), list)

def test_list_nonexistent_warehouse():
    response = client.get("/warehouses/99/items")
    assert response.status_code == 404

def test_in_stock_filter():
    client.post("/warehouses/2/items", json={
        "name": "Empty Box", "quantity": 0, "supplier_code": "EB-001"
    })
    client.post("/warehouses/2/items", json={
        "name": "Full Box", "quantity": 10, "supplier_code": "FB-001"
    })
    all_items = client.get("/warehouses/2/items").json()
    in_stock  = client.get("/warehouses/2/items?in_stock_only=true").json()

    assert len(in_stock) < len(all_items)
    assert all(item["quantity"] > 0 for item in in_stock)
```

---

# Level 3 Solutions

## Solution 3a & 3b: Create Endpoint + Quantity Constraint

```python
# schemas/inventory.py
from pydantic import BaseModel, Field

class ItemCreate(BaseModel):
    name: str
    quantity: int = Field(ge=0)   # 3b: ge=0 = "greater than or equal to 0"
    supplier_code: str

class ItemResponse(BaseModel):
    id: int
    name: str
    quantity: int
    warehouse_id: int
    # supplier_code: absent — response_model filters it
```

```python
# routers/inventory.py
from fastapi import APIRouter, HTTPException
from schemas.inventory import ItemCreate, ItemResponse

router = APIRouter(prefix="/warehouses", tags=["Inventory"])

warehouses_db: dict[int, dict] = {
    1: {"id": 1, "name": "North Warehouse"},
    2: {"id": 2, "name": "South Warehouse"},
}
items_db: dict[int, dict] = {}
_next_item_id = 1

@router.post("/{warehouse_id}/items", status_code=201, response_model=ItemResponse)
async def create_item(warehouse_id: int, item: ItemCreate):
    global _next_item_id

    # Constraint: warehouse must exist
    if warehouse_id not in warehouses_db:
        raise HTTPException(
            status_code=404,
            detail=f"Warehouse {warehouse_id} not found"
        )

    # Constraint: no duplicate names (case-insensitive) within same warehouse
    for existing in items_db.values():
        if (existing["warehouse_id"] == warehouse_id
                and existing["name"].lower() == item.name.lower()):
            raise HTTPException(
                status_code=409,
                detail=f"Item '{item.name}' already exists in this warehouse"
            )

    new_item = {
        "id": _next_item_id,
        "name": item.name,
        "quantity": item.quantity,
        "supplier_code": item.supplier_code,  # In the dict — stripped by response_model
        "warehouse_id": warehouse_id,
    }
    items_db[_next_item_id] = new_item
    _next_item_id += 1
    return new_item  # response_model=ItemResponse strips supplier_code automatically
```

```python
# main.py
from fastapi import FastAPI
from routers import inventory

app = FastAPI()
app.include_router(inventory.router)
```

**Why `Field(ge=0)` gives 422 instead of needing an `if` check:**
`ge=0` is a Pydantic constraint: "greater than or equal to 0." When FastAPI receives the request body, it passes it to `ItemCreate` for validation. `quantity=-5` fails the `ge=0` constraint — Pydantic raises a `ValidationError` before your function body ever runs. FastAPI converts that `ValidationError` into a 422 response automatically. This is Pydantic doing the validation work that you were forbidden from doing manually.

---

## Solution 3c: List Endpoint

```python
# Add to routers/inventory.py

@router.get("/{warehouse_id}/items", response_model=list[ItemResponse])
async def list_items(
    warehouse_id: int,
    in_stock_only: bool = False,   # Query param: not in path template, has default
):
    if warehouse_id not in warehouses_db:
        raise HTTPException(
            status_code=404,
            detail=f"Warehouse {warehouse_id} not found"
        )

    warehouse_items = [
        item for item in items_db.values()
        if item["warehouse_id"] == warehouse_id
    ]

    if in_stock_only:
        warehouse_items = [i for i in warehouse_items if i["quantity"] > 0]

    return warehouse_items
```

**Route order note:** Both `POST /{warehouse_id}/items` and `GET /{warehouse_id}/items` are dynamic routes with the same pattern, differentiated by HTTP method. FastAPI considers both method AND path when matching, so there is no conflict. If a `GET /warehouses/report` were added later, it would need to come before `GET /{warehouse_id}/items` in the router file to avoid being consumed by the wildcard — the same rule applies inside a router as in the main app.

**Why `quantity=0` is accepted by Pydantic but filtered by `in_stock_only`:** `ge=0` means quantity ≥ 0. Zero is a valid quantity (an item exists in the system with no stock). The `in_stock_only` filter is a business logic filter (`> 0`), not a validation constraint. These are different concerns: Pydantic validates structural correctness, the route filter applies business logic.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## Exercise 4: When `async def` Makes Things Worse

A developer reads the FastAPI documentation and this lecture, then decides that every single route in the application should be `async def`. Their reasoning: "async is modern, it's always faster, and being consistent is better than switching between `def` and `async def`."

**Your task:**

1. Describe a **specific, concrete scenario** where making a route `async def` produces measurably worse performance than regular `def`. The scenario must involve a realistic use case, not a contrived `time.sleep` example.

2. Write a minimal code snippet demonstrating the failure mode. Show what the bad version looks like and what the correct version looks like.

3. Explain the mechanism. Why is the bad version worse? Name what is being blocked and why that has a cascading effect.

4. State a decision rule a developer can apply to every new route function to choose correctly between `def` and `async def`.

---

# Level 4 Solution

## Solution 4: The CPU-Bound Route

**The Scenario:**
An API endpoint generates PDF reports from database data. It uses `reportlab`, a synchronous, CPU-intensive Python library. There is no async version of `reportlab` and no `await` anywhere in the generation logic.

```python
import io
from fastapi import FastAPI
from reportlab.pdfgen import canvas  # synchronous, CPU-bound

app = FastAPI()

# ❌ WRONG: async def with CPU-bound synchronous work
@app.post("/reports/pdf/bad")
async def generate_pdf_bad(title: str):
    buffer = io.BytesIO()
    c = canvas.Canvas(buffer)
    c.drawString(100, 750, title)
    # ... imagine 2 seconds of CPU-intensive layout calculation
    c.save()
    return {"size": buffer.tell()}

# ✅ CORRECT: def — FastAPI dispatches to thread pool
@app.post("/reports/pdf/good")
def generate_pdf_good(title: str):
    buffer = io.BytesIO()
    c = canvas.Canvas(buffer)
    c.drawString(100, 750, title)
    c.save()
    return {"size": buffer.tell()}
```

**The Mechanism:**

With `async def`, FastAPI runs the coroutine directly on the event loop's single thread. `reportlab` is entirely synchronous — it never calls `await`, never yields. The CPU grinds through 2 seconds of layout computation on the event loop thread. That thread is the **only** thread handling all incoming requests. Every other request — including trivial ones like `/health` or `/ping` — is queued and waiting. Ten concurrent report requests become sequential: 10 × 2s = 20 seconds total.

With `def`, FastAPI detects a synchronous function and runs it in a **thread pool** via `asyncio.run_in_executor`. The event loop thread dispatches the job and immediately picks up the next request. Ten concurrent requests are handled by ten thread pool threads. Total time drops to ~2 seconds regardless of concurrency.

```
10 concurrent requests to /reports/pdf/bad (async def + CPU work):
  T=0s:   Request 1 starts, freezes event loop
  T=2s:   Request 1 done, Request 2 starts, freezes event loop
  T=4s:   Request 2 done, Request 3 starts...
  T=20s:  All 10 done. Total: 20 seconds.

10 concurrent requests to /reports/pdf/good (def → thread pool):
  T=0s:   All 10 dispatched to thread pool simultaneously
  T≈2s:   All 10 done. Total: ~2 seconds.
```

**The Decision Rule:**

| Ask yourself | Answer | Use |
|---|---|---|
| Does my function use `await` anywhere? | Yes | `async def` |
| Does my function call an async library? (`asyncpg`, `httpx`, `aiofiles`) | Yes | `async def` |
| Does my function call a synchronous library that blocks? (`reportlab`, `PIL`, `requests`, `psycopg2`) | Yes | `def` |
| Does my function do CPU-intensive work? (image processing, compression, encryption) | Yes | `def` |
| My function is a pure transform with no I/O | Either works, `def` is simpler | `def` |

The core question: "Is there an `await` in this call chain?" If no, use `def`. FastAPI's thread pool is not a performance penalty — it is the correct tool for blocking work.

---

# LEVEL 5: THE TEACH BACK

## Exercise 5: Three Bugs, One Developer

A junior developer on your team pushes the following code and opens a pull request with this comment:

> *"I've been careful about optional parameters — I always write `Optional[type]` to mark them. But users keep hitting 422 errors saying fields are required. I think FastAPI has a bug.*
>
> *Also, my `GET /catalog/99` is supposed to return a 404, but it always returns 200 with some strange JSON blob instead of the error. I tried everything.*
>
> *And my `GET /catalog/featured` always gives me a 422 Unprocessable Entity. But that's a route I defined! Why is FastAPI rejecting 'featured' as invalid input?"*

```python
# catalog_api.py
from fastapi import FastAPI, HTTPException
from typing import Optional

app = FastAPI()

catalog: dict[int, dict] = {
    1: {"id": 1, "name": "Pen",      "price": 1.50},
    2: {"id": 2, "name": "Notebook", "price": 4.99},
}

@app.get("/catalog")
async def search_catalog(
    min_price: Optional[float],     # "marked as optional"
    max_price: Optional[float],     # "marked as optional"
    name_contains: str = "",
):
    results = list(catalog.values())
    if min_price:
        results = [p for p in results if p["price"] >= min_price]
    if max_price:
        results = [p for p in results if p["price"] <= max_price]
    if name_contains:
        results = [p for p in results if name_contains.lower() in p["name"].lower()]
    return results

@app.get("/catalog/{product_id}")
async def get_product(product_id: int):
    if product_id not in catalog:
        return HTTPException(status_code=404, detail="Product not found")
    return catalog[product_id]

@app.get("/catalog/featured")
async def get_featured():
    return [catalog[1], catalog[2]]
```

**Your task:** Act as the senior developer reviewing this PR. For each of the three bugs:

1. Explain **exactly what is happening** — trace through the mechanism step by step.
2. Explain **why the developer's mental model is wrong** — what did they assume that is not true?
3. Provide the **corrected code** for that specific bug only.
4. Give them a **rule to memorise** that prevents this class of mistake in the future.

Your goal is not to patch this specific code. Your goal is to explain the underlying principle clearly enough that they will never make each mistake again.

---

# Level 5 Solution

## Bug 1: `Optional[float]` ≠ Optional Parameter

**What is actually happening:**

`GET /catalog` without parameters triggers a 422 because FastAPI processes the function signature. It finds:

```python
min_price: Optional[float]   # No = sign anywhere
max_price: Optional[float]   # No = sign anywhere
```

There is no default value. FastAPI marks both as required. When the client sends no query parameters, FastAPI cannot populate them and rejects the request with a 422 — which is the correct behaviour given the declaration.

**Why the mental model is wrong:**

The developer conflated two separate concepts:
- `Optional[float]` is a **type declaration**. It tells Python: "the value of this variable is allowed to be either a `float` or `None`." This is purely about what value is permissible after it arrives.
- A default value (`= None`, `= 0.0`, etc.) is a **routing declaration**. It tells FastAPI: "if the client doesn't send this parameter, use this default instead." This is about whether the client must send it at all.

`Optional[float]` says nothing about whether the client must send the parameter. It only says that if the client sends `null` or nothing is provided after a default exists, the type system will accept it.

**Fix:**

```python
@app.get("/catalog")
async def search_catalog(
    min_price: Optional[float] = None,   # = None makes it optional
    max_price: Optional[float] = None,   # = None makes it optional
    name_contains: str = "",
):
    ...
```

**The rule to memorise:**
> The `=` sign is the gatekeeper. Default value = optional. No default value = required. The type annotation (`Optional`, `str`, `int`) describes the shape of the value after it arrives. The `=` decides whether it must arrive at all.

---

## Bug 2: `return HTTPException(...)` vs `raise HTTPException(...)`

**What is actually happening:**

When `GET /catalog/99` is called, `99 not in catalog` is `True`, so this line runs:

```python
return HTTPException(status_code=404, detail="Product not found")
```

`HTTPException(status_code=404, detail="...")` is Python object construction. It creates an instance of the `HTTPException` class and stores the attributes `status_code=404` and `detail="Product not found"`. The `return` statement hands this object to FastAPI's response serialiser. FastAPI sees a Python object, converts it to a JSON-compatible dictionary, wraps it in a `JSONResponse` with HTTP status 200, and sends it. The client receives:

```
HTTP/1.1 200 OK
{"status_code": 404, "detail": "Product not found"}
```

Status 200. Not 404. The number 404 is data inside the JSON body, not a protocol signal.

**Why the mental model is wrong:**

The developer assumed that creating and returning an `HTTPException` is enough to trigger the error-handling system. But `HTTPException` is only special when it is *raised*. `raise` activates Python's exception propagation mechanism. FastAPI registers an exception handler specifically for `HTTPException`. When one propagates, that handler intercepts it, reads `status_code`, constructs a real HTTP error response, and sends it. `return` never reaches the exception handler — it goes straight to the serialiser, which treats it as ordinary data.

**Fix:**

```python
@app.get("/catalog/{product_id}")
async def get_product(product_id: int):
    if product_id not in catalog:
        raise HTTPException(status_code=404, detail="Product not found")  # raise!
    return catalog[product_id]
```

**The rule to memorise:**
> `HTTPException` is an exception. Exceptions are raised, not returned. `return HTTPException(...)` is a silent bug: it produces a 200 response containing the exception's attributes as JSON. Always write `raise HTTPException(...)`. If you find yourself writing `return HTTPException`, stop — something is wrong.

---

## Bug 3: Static Routes Must Come Before Dynamic Routes

**What is actually happening:**

`GET /catalog/featured` triggers a 422. Here is why. When the request arrives, FastAPI walks its route list top-to-bottom:

```
Route 1: GET /catalog        → path is "/catalog/featured", doesn't match "/catalog". Skip.
Route 2: GET /catalog/{product_id}  → Does "featured" match {product_id}? YES, it's a wildcard.
         product_id is typed as int.
         FastAPI attempts: int("featured") → fails with ValueError.
         FastAPI returns 422 Unprocessable Entity.
         STOP. Route 3 is never checked.

Route 3: GET /catalog/featured   → Never reached.
```

The dynamic route at position 2 **claims** `"featured"` as its `product_id` value, then fails the type conversion. The static route at position 3 is permanently shadowed and unreachable.

**Why the mental model is wrong:**

The developer assumed FastAPI looks at all routes and picks the "best match" — the most specific one. It does not. FastAPI matches top-to-bottom and stops at the first route whose path template matches the URL structure, before considering whether type conversion will succeed. `/catalog/{product_id}` matches `/catalog/featured` structurally (one dynamic segment). The fact that `"featured"` cannot be converted to `int` does not cause FastAPI to skip this route and try the next — it causes FastAPI to return a 422 and stop.

**Fix:**

Place the static route before the dynamic route:

```python
@app.get("/catalog/featured")       # Static first — exact match, not a wildcard
async def get_featured():
    return [catalog[1], catalog[2]]

@app.get("/catalog/{product_id}")   # Dynamic last — wildcard after exact matches
async def get_product(product_id: int):
    if product_id not in catalog:
        raise HTTPException(status_code=404, detail="Product not found")
    return catalog[product_id]
```

**The rule to memorise:**
> Register routes in order from most specific to most general. Static literal paths (`/catalog/featured`) before dynamic wildcard paths (`/catalog/{id}`). This is identical to exception handling: `except SpecificError:` before `except Exception:`. A catch-all placed first consumes everything; it must come last.