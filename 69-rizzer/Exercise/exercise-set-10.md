# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "Error handling in web APIs is just returning descriptive dicts. `async def` makes everything inside it non-blocking. `Depends()` is a fancy way of calling a helper function upfront. Exception handlers are just neater try/except blocks attached to individual endpoints."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | `return {"error": "..."}` signals a failure to the HTTP client | The status code is 200 unless explicitly set; the body content is invisible to `response.ok` |
| 2 | `Depends(get_pagination)` calls `get_pagination` when the app starts | `Depends()` is metadata — FastAPI calls the function **per request**, not at definition time |
| 3 | `async def` makes all code inside it non-blocking | `async def` declares a coroutine; synchronous blocking calls inside still freeze the event loop |
| 4 | The same `Depends(fn)` referenced twice in one endpoint calls `fn` twice | FastAPI caches dependency results within a single request — `fn` runs once, result is reused |
| 5 | `@app.exception_handler(MyError)` only handles errors raised inside the decorated endpoint | It catches `MyError` from **anywhere** in the call stack — endpoint, service, dependency — for every request |
| 6 | Cleanup code after `yield` in a yield dependency always runs | Without `try/finally`, cleanup is skipped when an exception propagates through the generator |
| 7 | Custom exception classes are unnecessary boilerplate when HTTPException works fine | `HTTPException` couples business logic to HTTP; domain exceptions let service code be reused in CLI tools, background jobs, and tests without a web context |

---

## The Main Purpose (Essence)

**Core Insight:** Error handling and dependency injection both solve the same meta-problem: **eliminating scattered, inconsistent handling of cross-cutting concerns**. Without them, every endpoint independently duplicates validation, error formatting, resource management, and authentication. With them, each endpoint expresses only what it uniquely owns.

| Topic | Without It | With It |
|-------|-----------|---------|
| `raise HTTPException` | Return dicts with 200; clients cannot detect failures via status code | Status codes carry semantic meaning; `response.ok` is accurate |
| Custom exception classes | `HTTPException` buried in service layer; untestable without web context | Pure Python domain exceptions; web layer translates to HTTP at the boundary |
| Exception handlers | Each endpoint has try/except; error shape drifts per developer | Register once globally; guaranteed consistent error shape for every endpoint |
| `Depends()` | Validation/auth copy-pasted into every endpoint | Declare once, inject everywhere; one change propagates automatically |
| Yield dependencies | Manual setup/teardown; cleanup silently skipped on exceptions | Guaranteed cleanup via `try/finally`; same semantic as context managers from Week 1 L2 |
| Dependency caching | Same expensive operation runs N times per request | Per-request cache: one call, result shared by all consumers within that request |
| `def` vs `async def` in `Depends()` | Blocking I/O in `async def` freezes the event loop | `def` runs in a threadpool automatically; event loop stays free |

**The Key Distinctions:**
- `raise HTTPException(...)` is a **control flow statement** that terminates execution, not a return value
- `Depends(fn)` is **metadata**, not a function call — FastAPI reads it and calls `fn` at request time
- A yield dependency is a **context manager expressed as a generator** — the same `try/finally` guarantee from Week 1 L2, consumed by FastAPI instead of a `with` statement

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| `return {"error": "..."}` = failure | Client detects failure via `response.ok` | `response.ok` is `True`; 200 status sent | HTTP status code, not body content, determines `ok` |
| `Depends(fn)` runs at startup | `call_log` has entries before first request | `call_log` is empty until first request | `Depends()` stores a reference; FastAPI resolves it per-request |
| `async def` = non-blocking | Two `async def` dependencies perform identically | `time.sleep()` inside `async def` freezes the event loop; `await asyncio.sleep()` does not | `async def` is a declaration, not a magic wrapper |
| Same `Depends(fn)` twice = two calls | Two parameters get different values | Both get the same cached value from one call | Per-request dependency cache (caching by function identity) |
| Handler catches all raises of `MyError` | Endpoint that re-raises as `HTTPException` uses custom handler | Re-raised `HTTPException` hits built-in handler, not custom one | Exception type in flight changed at re-raise |
| Yield cleanup always runs | Connection closes even on error | Without `try/finally`, cleanup code after `yield` is skipped when endpoint raises | Generator abandonment on exception propagation |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Client That Thinks Everything Is Fine

```python
from fastapi import FastAPI
from fastapi.testclient import TestClient

app = FastAPI()

inventory: dict[int, dict] = {
    1: {"id": 1, "name": "Widget", "stock": 10},
    2: {"id": 2, "name": "Gadget", "stock": 0},
}


@app.get("/items/{item_id}")
async def get_item(item_id: int):
    if item_id not in inventory:
        return {"error": f"Item {item_id} not found"}
    item = inventory[item_id]
    if item["stock"] == 0:
        return {"error": f"Item '{item['name']}' is out of stock"}
    return item


@app.post("/orders")
async def create_order(item_id: int, quantity: int):
    if item_id not in inventory:
        return {"error": "Item not found"}
    if inventory[item_id]["stock"] < quantity:
        return {"error": "Insufficient stock"}
    inventory[item_id]["stock"] -= quantity
    return {"order": "created", "item_id": item_id, "quantity": quantity}


# A frontend developer wrote this client logic:
client = TestClient(app)


def place_order(item_id: int, quantity: int) -> str:
    response = client.post(f"/orders?item_id={item_id}&quantity={quantity}")
    if response.ok:
        return "Order placed successfully!"
    else:
        return f"Order failed: {response.json()}"


results = [
    place_order(1, 5),   # Case 1: Valid item with sufficient stock
    place_order(2, 1),   # Case 2: Valid item but out of stock
    place_order(99, 1),  # Case 3: Non-existent item
]
for r in results:
    print(r)
```

**Questions:**

1. **Predict the output.** What does each `place_order` call print? Give the exact string for all three cases.

2. **Explain the danger.** For Cases 2 and 3, what is `response.status_code`? Why is `response.ok` True for failures? How does this make the frontend developer's `if response.ok` check unreliable?

3. **Mental Trace.** The frontend team builds a monitoring dashboard that counts failed orders by checking `if not response.ok`. After a week of production traffic with 1,000 orders attempted, how many failures does the dashboard report? What is silently happening to the business data?

---

## Exercise 1.2: The Punctual Dependency

```python
from fastapi import FastAPI, Depends
from fastapi.testclient import TestClient
import uuid

app = FastAPI()
call_log: list[str] = []


def get_request_id() -> str:
    request_id = str(uuid.uuid4())[:8]
    call_log.append(f"get_request_id called → {request_id}")
    return request_id


@app.get("/ping")
async def ping(req_id: str = Depends(get_request_id)):
    return {"pong": True, "request_id": req_id}


@app.get("/status")
async def status(req_id: str = Depends(get_request_id)):
    return {"ok": True, "request_id": req_id}


print(f"BEFORE any request — call_log: {call_log}")

client = TestClient(app)
client.get("/ping")
client.get("/ping")
client.get("/status")

print(f"AFTER 3 requests — call_log length: {len(call_log)}")
print(f"call_log: {call_log}")
```

**Questions:**

1. **Predict the output.** What is in `call_log` before any requests? After 3 requests, what is `len(call_log)`? What do the entries look like?

2. **Explain why.** A student argued: "`Depends(get_request_id)` is evaluated when the `@app.get` decorator runs at import time, so `get_request_id` must be called at that moment." Why is this argument wrong? At exactly what moment does `get_request_id` execute?

3. **Extend the mental model.** Now consider this endpoint without running it:

```python
@app.get("/compare")
async def compare(
    id_a: str = Depends(get_request_id),
    id_b: str = Depends(get_request_id),
):
    return {"id_a": id_a, "id_b": id_b, "same": id_a == id_b}
```

How many times does `get_request_id` get called per request to `/compare`? Are `id_a` and `id_b` the same value? Why?

---

## Exercise 1.3: The Async Impostor

```python
from fastapi import FastAPI, Depends
import time
import asyncio

app = FastAPI()


# Dependency A: blocking call inside async def
async def get_config_a() -> dict:
    time.sleep(2)           # Simulates reading from a slow config store
    return {"version": "v1", "debug": True}


# Dependency B: awaitable call inside async def
async def get_config_b() -> dict:
    await asyncio.sleep(2)  # Simulates async I/O
    return {"version": "v1", "debug": True}


@app.get("/a")
async def endpoint_a(config: dict = Depends(get_config_a)):
    return config


@app.get("/b")
async def endpoint_b(config: dict = Depends(get_config_b)):
    return config
```

**Questions:**

1. **Predict the behavior for a single client.** Both endpoints return identical data and take ~2 seconds. From one client's perspective, they look the same. Which one is dangerous in production under concurrent load? Why?

2. **Explain the mechanism.** A student argues: "Both functions are `async def`, so FastAPI treats them identically. They're equivalent." Explain precisely what `async def` actually guarantees, and what it absolutely does NOT guarantee about the code inside it.

3. **Propose the fix.** `get_config_a` reads from a synchronous config store — you cannot change the library. Write only the corrected function signature (the `def` line, not the body). Explain in one sentence why changing only the signature — without touching the body — fixes the problem.

---

# Level 1 Solutions

## Solution 1.1: The Client That Thinks Everything Is Fine

**1. Predicted output:**

```
Order placed successfully!
Order placed successfully!
Order placed successfully!
```

All three calls return "Order placed successfully!" — including the two failure cases.

**2. The danger:**

For all three cases, `response.status_code` is `200`. FastAPI sees `return {"error": "..."}` as a successful endpoint returning a JSON body that happens to contain a key named "error." HTTP has no rule that a body with an "error" key means failure. The status code is the contract, and it says 200. `response.ok` is `True` for any status in the 200–299 range, so the conditional `if response.ok` evaluates to `True` for Cases 2 and 3. The client's error-handling branch (`else`) never executes.

**3. The monitoring dashboard:**

The failure count is **zero** — permanently. Every response carries a 200 status code, so `if not response.ok` is never `True`. After 1,000 order attempts, the dashboard shows a perfect success rate. Out-of-stock errors and invalid item lookups are silently treated as successful order placements in the monitoring system. The business has no visibility into failure rates, and stock levels may be incorrectly decremented or customers may receive false "Order placed!" confirmations. This is not a cosmetic problem — it is an operational blind spot.

**The fix:** Replace every `return {"error": ...}` with `raise HTTPException(status_code=..., detail=...)`. Now `response.ok` accurately reflects whether the operation succeeded.

---

## Solution 1.2: The Punctual Dependency

**1. Predicted output:**

```
BEFORE any request — call_log: []
AFTER 3 requests — call_log length: 3
call_log: [
    'get_request_id called → a1b2c3d4',
    'get_request_id called → e5f6a7b8',
    'get_request_id called → c9d0e1f2'
]
```

Before any request, `call_log` is empty. After 3 requests, it contains exactly 3 entries — one per request.

**2. Why the student's argument is wrong:**

`@app.get("/ping")` is a decorator that runs at import time — it registers the route. But `Depends(get_request_id)` inside the function signature does not call `get_request_id`. It creates a `Depends` object that stores a *reference* to `get_request_id`. FastAPI reads this object when processing an incoming request and calls `get_request_id()` then — not before.

The analogy from the lecture: `Depends(get_request_id)` is writing "call room service" on an order form. Writing it down is not calling room service. Room service is called only when a guest submits the form — i.e., when a request arrives.

**3. The `/compare` caching question:**

`get_request_id` is called **once** per request to `/compare`. Both `id_a` and `id_b` hold the **same value**. FastAPI's per-request dependency cache sees that `get_request_id` has already been resolved during this request and reuses the cached result for the second `Depends(get_request_id)`. A new request generates a fresh `request_id`, but within one request's lifetime, identical `Depends(fn)` references share one cached result.

---

## Solution 1.3: The Async Impostor

**1. Which is dangerous:**

`get_config_a` (using `time.sleep`) is dangerous in production. `time.sleep(2)` inside `async def` blocks the **event loop** for 2 full seconds. During those 2 seconds, zero other requests in the entire application can be processed. With 50 concurrent users, each request queues behind the blocked event loop, producing cascading latency.

`get_config_b` (using `await asyncio.sleep(2)`) releases control back to the event loop during the wait, allowing other coroutines to progress concurrently.

**2. Why the student is wrong:**

`async def` is a **coroutine declaration**. It tells Python that this function may contain `await` points where control is voluntarily yielded back to the event loop. It does NOT make all code inside it non-blocking. `time.sleep(2)` is a synchronous blocking call — it holds the CPU and the event loop completely, regardless of whether it appears inside `async def` or `def`. The `async def` keyword has zero effect on synchronous code within it.

`await asyncio.sleep(2)` is genuinely non-blocking because `asyncio.sleep` creates a coroutine that explicitly suspends itself and notifies the event loop to schedule other tasks in the meantime.

**3. The correct fix:**

```python
def get_config_a() -> dict:   # Changed: async def → def
    time.sleep(2)
    return {"version": "v1", "debug": True}
```

Changing `async def` to `def` tells FastAPI: "this is a synchronous dependency." FastAPI detects plain `def` and runs it in a **threadpool** via `asyncio.run_in_executor`. The blocking `time.sleep` now runs in a separate OS thread, leaving the event loop entirely free to serve other requests concurrently.

---

# LEVEL 2: DEBUG, FIX & VERIFY

## Exercise 2.1: The Exception Handler That Isn't

**Context:** A developer built a task API with what they believe is a complete, globally consistent error handling system. The code runs without crashing and returns 404s. But after a code review, a teammate points out that clients are receiving two completely different error shapes depending on which endpoint they call.

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI()

tasks_db: dict[int, dict] = {
    1: {"id": 1, "title": "Buy milk", "completed": False},
}


class TaskNotFoundError(Exception):
    def __init__(self, task_id: int):
        self.task_id = task_id
        super().__init__(f"Task {task_id} not found")


@app.exception_handler(TaskNotFoundError)
async def task_not_found_handler(request: Request, exc: TaskNotFoundError):
    return JSONResponse(
        status_code=404,
        content={"error": "not_found", "detail": str(exc), "task_id": exc.task_id}
    )


def get_task_from_db(task_id: int) -> dict:
    if task_id not in tasks_db:
        raise TaskNotFoundError(task_id)
    return tasks_db[task_id]


@app.get("/tasks/{task_id}")
async def get_task(task_id: int):
    try:
        return get_task_from_db(task_id)
    except TaskNotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))


@app.delete("/tasks/{task_id}", status_code=204)
async def delete_task(task_id: int):
    get_task_from_db(task_id)   # Raises TaskNotFoundError if missing
    del tasks_db[task_id]
```

**Task 2a: Identify the Flaw**

Mentally run both endpoints against `task_id=999`. What are the two different JSON response bodies? The developer intended ONE consistent shape from the global exception handler. Why are they different?

**Task 2b: Explain the Mechanism**

1. In `get_task`, `TaskNotFoundError` is caught and re-raised as `HTTPException`. Which handler processes the exception that ultimately reaches FastAPI — `@app.exception_handler(TaskNotFoundError)` or FastAPI's built-in `HTTPException` handler?
2. In `delete_task`, no try/except exists. Which handler processes the `TaskNotFoundError` that bubbles up?
3. Trace the exception flow for both endpoints. At exactly what point does each exception reach a handler?

**Task 2c: Instrument to Prove Diagnosis**

Add `print` statements inside both handlers to prove which one fires for each endpoint:

```python
@app.exception_handler(TaskNotFoundError)
async def task_not_found_handler(request: Request, exc: TaskNotFoundError):
    print(f"DEBUG TaskNotFoundError handler: path={request.url.path}")
    # ... rest unchanged


from fastapi.exception_handlers import http_exception_handler as _default_http
from fastapi import HTTPException as _HTTPExc

@app.exception_handler(_HTTPExc)
async def debug_http_handler(request: Request, exc: _HTTPExc):
    print(f"DEBUG HTTPException handler: status={exc.status_code}, path={request.url.path}")
    return await _default_http(request, exc)
```

Write out the expected console output for `GET /tasks/999` and `DELETE /tasks/999`.

**Task 2d: Fix the Code**

Fix `get_task` so that:
- Both `GET` and `DELETE` return the same error shape
- `get_task` contains no try/except block
- `@app.exception_handler(TaskNotFoundError)` is the single source of error formatting

Show only the changed function.

**Task 2e: Write Verification Test**

Write a test that **fails on the original code** and **passes on your fix**:

```python
from fastapi.testclient import TestClient
client = TestClient(app)

def test_consistent_error_shape():
    """Both endpoints must return identical error shapes for a missing task."""
    get_resp = client.get("/tasks/999")
    delete_resp = client.delete("/tasks/999")

    # Your assertions here:
    # 1. Both return 404
    # 2. Both have the same top-level keys
    # 3. Both have "error" == "not_found"
    # 4. Both have "task_id" == 999
```

---

## Exercise 2.2: The Resource That Never Closes

**Context:** A developer implemented a yield dependency for database connection management. Responses return correctly during normal traffic. But after a sustained load test, memory keeps climbing and the connection pool is exhausted. A colleague suspects a resource leak.

```python
from fastapi import FastAPI, Depends
from fastapi.testclient import TestClient
from typing import Generator

app = FastAPI()

active_connections: list[str] = []
_conn_counter = 0


class DBConnection:
    def __init__(self):
        global _conn_counter
        _conn_counter += 1
        self.id = f"conn-{_conn_counter}"
        self.closed = False
        active_connections.append(self.id)
        print(f"[{self.id}] Opened. Active: {len(active_connections)}")

    def close(self):
        self.closed = True
        active_connections.remove(self.id)
        print(f"[{self.id}] Closed. Active: {len(active_connections)}")

    def query(self, task_id: int) -> dict:
        if self.closed:
            raise RuntimeError("Query on closed connection!")
        if task_id > 100:
            raise ValueError(f"Task {task_id} exceeds valid range")
        return {"id": task_id, "title": f"Task {task_id}"}


def get_db() -> Generator[DBConnection, None, None]:
    conn = DBConnection()
    yield conn
    conn.close()    # Is this guaranteed to run?


@app.get("/tasks/{task_id}")
async def get_task(task_id: int, db: DBConnection = Depends(get_db)):
    return db.query(task_id)


client = TestClient(app, raise_server_exceptions=False)

print("=== Request 1: Valid task ===")
r1 = client.get("/tasks/5")
print(f"Active after request 1: {len(active_connections)}")

print("\n=== Request 2: Task that triggers an exception ===")
r2 = client.get("/tasks/999")
print(f"Active after request 2: {len(active_connections)}")

print("\n=== Request 3: Another valid task ===")
r3 = client.get("/tasks/10")
print(f"Active after request 3: {len(active_connections)}")
```

**Task 2a: Identify the Flaw**

1. Predict how many connections are in `active_connections` after all three requests.
2. Which specific request causes the leak? Why?
3. What does `active_connections` contain after all three requests have completed?

**Task 2b: Explain the Mechanism**

1. In the `get_db` generator, where is `conn.close()` positioned relative to `yield`?
2. When `db.query(999)` raises `ValueError`, what happens to the generator's execution?
3. Why is this the exact same bug as a `try` block without `finally` from Week 1 Lecture 2?

**Task 2c: Instrument to Prove Diagnosis**

Add a global counter that tracks whether `close()` is ever called:

```python
close_call_count = 0

class DBConnection:
    def close(self):
        global close_call_count
        close_call_count += 1
        # ... rest of close() unchanged
```

Predict `close_call_count` after all 3 requests on the **original broken code**. Then predict what it should be on **fixed code**.

**Task 2d: Fix the Code**

Fix `get_db` so `conn.close()` is **guaranteed** to run after every request, whether the endpoint succeeds or raises. Show only the changed function.

**Task 2e: Write Verification Test**

```python
def test_connection_always_closed():
    """Connection must close even when the endpoint raises an exception."""
    initial_active = len(active_connections)

    client.get("/tasks/999")   # Triggers ValueError inside the endpoint

    # What should len(active_connections) equal?
    assert len(active_connections) == ???
```

Fill in the assertion and explain why the original code fails this test.

---

# Level 2 Solutions

## Solution 2.1: The Exception Handler That Isn't

**Task 2a: The Two Different Shapes**

`GET /tasks/999`:
```json
{"detail": "Task 999 not found"}
```

`DELETE /tasks/999`:
```json
{"error": "not_found", "detail": "Task 999 not found", "task_id": 999}
```

The developer's custom handler produces the second shape. FastAPI's built-in handler produces the first.

**Task 2b: The Mechanism**

1. In `get_task`, `TaskNotFoundError` is caught and re-raised as `HTTPException`. Once re-raised, the exception **in flight** is now an `HTTPException` instance — not a `TaskNotFoundError`. FastAPI's handler registry does not see `TaskNotFoundError`; it sees `HTTPException` and routes it to the built-in handler, producing `{"detail": "..."}`.

2. In `delete_task`, `TaskNotFoundError` bubbles up uncaught. FastAPI checks its registry, finds `@app.exception_handler(TaskNotFoundError)`, and routes it there, producing the structured `{"error": "not_found", ...}` shape.

3. Exception flow:

```
GET /tasks/999:
  get_task() → get_task_from_db() raises TaskNotFoundError
             → try/except catches it
             → re-raises as HTTPException          ← type changes here
             → FastAPI built-in HTTPException handler fires
             → {"detail": "Task 999 not found"}

DELETE /tasks/999:
  delete_task() → get_task_from_db() raises TaskNotFoundError
                → bubbles up (no try/except)
                → @app.exception_handler(TaskNotFoundError) fires
                → {"error": "not_found", "detail": "...", "task_id": 999}
```

**Task 2c: Instrumented Output**

```
# GET /tasks/999:
DEBUG HTTPException handler: status=404, path=/tasks/999

# DELETE /tasks/999:
DEBUG TaskNotFoundError handler: path=/tasks/999
```

**Task 2d: Fixed Code**

```python
@app.get("/tasks/{task_id}")
async def get_task(task_id: int):
    return get_task_from_db(task_id)
    # TaskNotFoundError bubbles up directly to @app.exception_handler — no try/except needed
```

By removing the catch-and-reraise, `TaskNotFoundError` reaches FastAPI in its original form. The custom handler now fires for both endpoints, guaranteeing one consistent error shape everywhere.

**Task 2e: Verification Test**

```python
def test_consistent_error_shape():
    get_resp = client.get("/tasks/999")
    delete_resp = client.delete("/tasks/999")

    assert get_resp.status_code == 404
    assert delete_resp.status_code == 404

    get_body = get_resp.json()
    delete_body = delete_resp.json()

    # Same top-level keys
    assert set(get_body.keys()) == set(delete_body.keys())

    # Both carry the structured error fields
    assert get_body["error"] == "not_found"
    assert delete_body["error"] == "not_found"
    assert get_body["task_id"] == 999
    assert delete_body["task_id"] == 999
```

On the original code, `get_resp.json()` is `{"detail": "..."}` — accessing `get_body["error"]` raises `KeyError` and the test fails. On the fixed code, both responses match the custom handler's shape and all assertions pass.

---

## Solution 2.2: The Resource That Never Closes

**Task 2a: Active Connections After All Three Requests**

`len(active_connections)` is **1** after all three requests. The leaked connection is from Request 2.

- **Request 1** (task_id=5): Opens conn-1, query succeeds, generator resumes after yield, `conn.close()` runs. Active: 0.
- **Request 2** (task_id=999): Opens conn-2, `db.query(999)` raises `ValueError`. FastAPI catches the exception from the endpoint. The generator is abandoned — Python throws the exception into the generator at the yield point, but without `try/finally`, the code after `yield` is never reached. `conn.close()` is skipped. Active: 1.
- **Request 3** (task_id=10): Opens conn-3, query succeeds, `conn.close()` runs. Active: still 1 (conn-2 remains leaked).

**Task 2b: The Mechanism**

1. `conn.close()` is placed **after `yield`** with no `try/finally`. It executes only if the generator resumes normally after yielding control.
2. When `db.query(999)` raises `ValueError`, FastAPI propagates the exception into the generator via `generator.throw()`. Without a surrounding `try/finally`, Python has no cleanup handler — the code after `yield` is simply never reached.
3. This is identical to:

```python
conn = create_connection()
result = conn.query(task_id)    # May raise!
conn.close()                    # Skipped if query raises — connection leaked
```

This is the exact bug `try/finally` (and context managers) exist to solve, as covered in Week 1 L2.

**Task 2c: Instrumented Count**

On **original broken code**: `close_call_count = 2` (Requests 1 and 3 close; Request 2 does not).

On **fixed code**: `close_call_count = 3` (all three requests close their connection).

**Task 2d: Fixed Code**

```python
def get_db() -> Generator[DBConnection, None, None]:
    conn = DBConnection()
    try:
        yield conn
    finally:
        conn.close()    # Guaranteed — error or not
```

**Task 2e: Verification Test**

```python
def test_connection_always_closed():
    initial_active = len(active_connections)
    client.get("/tasks/999")    # Triggers ValueError inside endpoint
    assert len(active_connections) == initial_active    # No leak
```

On the original code, Request 2 leaks conn-2, so `len(active_connections)` is `initial_active + 1` — the assertion fails. On the fixed code, `finally` runs and `conn.close()` removes the connection, returning `active_connections` to `initial_active` — the assertion passes.

---

# LEVEL 2.5: EXTEND EXISTING CODE

## The Integration Challenge

**Given:** The following is working, tested code. All three tests currently pass. Your task is to add three extensions in sequence. **Existing tests must pass after each extension.**

```python
# dependencies.py
from typing import Annotated
from dataclasses import dataclass
from fastapi import Depends, Query
import structlog
import uuid

logger = structlog.get_logger()


@dataclass
class PaginationParams:
    skip: int
    limit: int


async def get_pagination(
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=10, ge=1, le=100),
) -> PaginationParams:
    return PaginationParams(skip=skip, limit=limit)


async def get_request_logger() -> structlog.stdlib.BoundLogger:
    return logger.bind(request_id=str(uuid.uuid4())[:8])


PaginationDep = Annotated[PaginationParams, Depends(get_pagination)]
LogDep = Annotated[structlog.stdlib.BoundLogger, Depends(get_request_logger)]
```

```python
# main.py
from fastapi import FastAPI
from dependencies import PaginationDep, LogDep

app = FastAPI()

tasks_db: dict[int, dict] = {
    i: {"id": i, "title": f"Task {i}", "completed": i % 2 == 0, "priority": i % 3}
    for i in range(1, 51)
}


@app.get("/tasks")
async def list_tasks(pagination: PaginationDep, log: LogDep):
    log.info("listing_tasks", skip=pagination.skip, limit=pagination.limit)
    all_tasks = list(tasks_db.values())
    return all_tasks[pagination.skip: pagination.skip + pagination.limit]
```

```python
# test_main.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)


def test_default_pagination():
    resp = client.get("/tasks")
    assert resp.status_code == 200
    assert len(resp.json()) == 10


def test_custom_pagination():
    resp = client.get("/tasks?skip=5&limit=3")
    assert resp.status_code == 200
    assert len(resp.json()) == 3
    assert resp.json()[0]["id"] == 6


def test_invalid_pagination():
    resp = client.get("/tasks?skip=-1")
    assert resp.status_code == 422
```

---

### Extension 1: Add Filtering (Moderate)

Add a `TaskFiltersDep` dependency that allows filtering by `completed` status and `priority` level.

Requirements:
- `completed`: optional boolean query param (default: `None` = no filter applied)
- `priority`: optional integer query param, valid values 0, 1, or 2 (default: `None` = no filter applied)
- Filtering must occur **before** pagination — filter the full set first, then paginate the filtered result
- The endpoint signature adds exactly one new parameter

The following tests must pass after your extension (in addition to the original three):

```python
def test_filter_by_completed():
    resp = client.get("/tasks?completed=true&limit=100")
    tasks = resp.json()
    assert len(tasks) > 0
    assert all(t["completed"] is True for t in tasks)


def test_filter_by_priority():
    resp = client.get("/tasks?priority=1&limit=100")
    tasks = resp.json()
    assert len(tasks) > 0
    assert all(t["priority"] == 1 for t in tasks)


def test_filter_and_paginate():
    resp = client.get("/tasks?completed=false&skip=0&limit=2")
    tasks = resp.json()
    assert len(tasks) == 2
    assert all(t["completed"] is False for t in tasks)
```

---

### Extension 2: Add a Request Timing Gatekeeper (Harder)

Add a gatekeeper dependency `log_slow_request` that:
- Measures elapsed time from request start to request end
- Logs a `structlog` warning if elapsed time exceeds a threshold (default: 500ms): `log.warning("slow_request", elapsed_ms=..., path=...)`
- Injects **nothing** into the endpoint — it is a pure lifecycle observer
- Must be applied at the **router level**, not per-endpoint

Constraints:
- The `list_tasks` function signature must not gain a new parameter
- The gatekeeper must have access to the `Request` object (to read the path)
- The gatekeeper must run **code after the endpoint** (to measure total time) — think carefully about what kind of dependency this implies

The following test must pass:

```python
def test_gatekeeper_does_not_break_response():
    resp = client.get("/tasks?limit=5")
    assert resp.status_code == 200
    assert len(resp.json()) == 5
```

---

### Extension 3: Add a Sync Config Dependency That Caps Results (Hardest)

A new requirement: the maximum result size must be controlled by a config file `config.json`, read at request time using the standard `json` library (synchronous, blocking I/O).

```json
{"max_results": 25}
```

Requirements:
- Create a `get_config` dependency that reads `config.json`
- If a client requests `limit > config["max_results"]`, silently cap the effective limit — do not raise an error
- The `get_config` dependency must use the correct `def`/`async def` declaration for this I/O type
- `get_pagination` must chain through `ConfigDep` — it receives the config and applies the cap internally
- The endpoint signature must not change

The following test must pass:

```python
def test_limit_capped_by_config(tmp_path, monkeypatch):
    import json
    import dependencies  # your module

    config_file = tmp_path / "config.json"
    config_file.write_text(json.dumps({"max_results": 5}))
    monkeypatch.setattr(dependencies, "CONFIG_PATH", config_file)

    resp = client.get("/tasks?limit=100")
    assert resp.status_code == 200
    assert len(resp.json()) <= 5
```

---

# Level 2.5 Solutions

## Solution: Extension 1 — Filtering

```python
# In dependencies.py — add:
from typing import Optional

@dataclass
class TaskFilters:
    completed: Optional[bool]
    priority: Optional[int]


async def get_task_filters(
    completed: Optional[bool] = Query(default=None),
    priority: Optional[int] = Query(default=None, ge=0, le=2),
) -> TaskFilters:
    return TaskFilters(completed=completed, priority=priority)


TaskFiltersDep = Annotated[TaskFilters, Depends(get_task_filters)]
```

```python
# In main.py — updated endpoint:
from dependencies import PaginationDep, LogDep, TaskFiltersDep


@app.get("/tasks")
async def list_tasks(
    pagination: PaginationDep,
    log: LogDep,
    filters: TaskFiltersDep,
):
    log.info("listing_tasks", skip=pagination.skip, limit=pagination.limit)
    all_tasks = list(tasks_db.values())

    # Filter FIRST (before pagination)
    if filters.completed is not None:
        all_tasks = [t for t in all_tasks if t["completed"] == filters.completed]
    if filters.priority is not None:
        all_tasks = [t for t in all_tasks if t["priority"] == filters.priority]

    # Then paginate the filtered result
    return all_tasks[pagination.skip: pagination.skip + pagination.limit]
```

**Why filter before paginate:** Pagination's `skip` and `limit` must apply to the *filtered* total, not the full dataset. Paginating first and filtering second produces incorrect page boundaries — page 2 of "completed tasks" would include tasks that aren't completed.

---

## Solution: Extension 2 — Request Timing Gatekeeper

```python
# In dependencies.py — add:
import time
from fastapi import Request


async def log_slow_request(
    request: Request,
    log: LogDep,
    threshold_ms: float = 500.0,
) -> None:
    start = time.time()
    yield                                               # Endpoint runs here
    elapsed_ms = (time.time() - start) * 1000
    if elapsed_ms > threshold_ms:
        log.warning(
            "slow_request",
            elapsed_ms=round(elapsed_ms, 2),
            path=str(request.url.path),
        )
```

```python
# In main.py — apply at router level:
from fastapi import APIRouter, Depends
from dependencies import PaginationDep, LogDep, TaskFiltersDep, log_slow_request

router = APIRouter(dependencies=[Depends(log_slow_request)])


@router.get("/tasks")
async def list_tasks(pagination: PaginationDep, log: LogDep, filters: TaskFiltersDep):
    # Unchanged body
    ...


app.include_router(router)
```

**Key insight:** `log_slow_request` must be a **yield dependency** because it needs to execute code *after* the endpoint returns (to measure total elapsed time). A regular dependency runs only before the endpoint. The `yield` is what creates the "after endpoint" lifecycle hook. This is the same mechanism as the yield-based cleanup in Exercise 2.2 — here used for timing instead of resource cleanup.

---

## Solution: Extension 3 — Sync Config Dependency

```python
# In dependencies.py — add:
import json
from pathlib import Path

CONFIG_PATH = Path("config.json")


def get_config() -> dict:       # NOTE: def, NOT async def — blocking file I/O!
    with open(CONFIG_PATH) as f:
        return json.load(f)


ConfigDep = Annotated[dict, Depends(get_config)]


# Updated get_pagination to chain through get_config:
async def get_pagination(
    config: ConfigDep,          # FastAPI resolves get_config first
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=10, ge=1, le=100),
) -> PaginationParams:
    max_results = config.get("max_results", 100)
    effective_limit = min(limit, max_results)           # Silent cap
    return PaginationParams(skip=skip, limit=effective_limit)
```

**Why `def` and not `async def`:** `open()` and `json.load()` are synchronous blocking file I/O. Declaring the function as `def` tells FastAPI to run it in a **threadpool** via `asyncio.run_in_executor`. The blocking read happens in a separate thread, leaving the event loop free. Declaring it `async def` with blocking I/O inside would freeze the event loop — the same class of bug from Exercise 1.3.

**Why chaining:** `get_pagination` lists `ConfigDep` as a parameter. FastAPI resolves dependencies bottom-up: `get_config()` runs first (in threadpool), its result is injected as `config` into `get_pagination()`. The endpoint receives only `PaginationParams` — the config is completely invisible to it.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## Exercise 3: The Multi-Tenant Gatekeeper Chain

**Scenario:** You are building a public API platform. Three cross-cutting concerns must be enforced on all endpoints under `/api/v1`.

**Concern 1 — Tenant identification:** Every request must include an `X-Tenant-ID` header. Missing or empty header → 400. Tenant ID not in the approved list → 403.

**Concern 2 — Per-tenant rate limiting:** Each tenant may make at most 5 requests per 10-second window. Exceeding this → 429. Use an in-memory dict store.

**Concern 3 — Audit logging:** Every request must be logged with structlog, capturing: tenant ID (or "MISSING"), endpoint path, HTTP method, and timestamp. This must run **even for rejected requests** (rejected by tenant check or rate limiter).

**Constraints — none of these may be violated:**
- Audit logging must fire for **every** request, including ones rejected by Concern 1 or 2
- Tenant checking must run **before** rate limiting
- No endpoint function may have more than 2 parameters
- The audit logger must be a **gatekeeper** (injects nothing)
- All concerns must be expressed at the **router level**, not per-endpoint
- Dependency type aliases must be defined for all injectable values

```python
APPROVED_TENANTS = {"tenant-alpha", "tenant-beta", "tenant-gamma"}
```

**Provided test suite — all must pass:**

```python
from fastapi.testclient import TestClient
# Assume app and client are importable from your solution

def test_missing_tenant_returns_400():
    resp = client.get("/api/v1/data")
    assert resp.status_code == 400


def test_unknown_tenant_returns_403():
    resp = client.get("/api/v1/data", headers={"X-Tenant-ID": "tenant-evil"})
    assert resp.status_code == 403


def test_valid_tenant_gets_data():
    resp = client.get("/api/v1/data", headers={"X-Tenant-ID": "tenant-alpha"})
    assert resp.status_code == 200


def test_rate_limit_enforced():
    for _ in range(5):
        resp = client.get("/api/v1/data", headers={"X-Tenant-ID": "tenant-beta"})
        assert resp.status_code == 200
    sixth = client.get("/api/v1/data", headers={"X-Tenant-ID": "tenant-beta"})
    assert sixth.status_code == 429


def test_rate_limit_is_per_tenant():
    for _ in range(5):
        client.get("/api/v1/data", headers={"X-Tenant-ID": "tenant-gamma"})
    # tenant-alpha has a clean slate — must not be affected by gamma's usage
    resp = client.get("/api/v1/data", headers={"X-Tenant-ID": "tenant-alpha"})
    assert resp.status_code == 200
```

**Task 3a: Implement Core**

Implement the solution so `test_missing_tenant_returns_400`, `test_unknown_tenant_returns_403`, and `test_valid_tenant_gets_data` pass. Design the dependency graph and router structure before writing any code.

**Task 3b: Meet the Rate Limiting Constraint**

Extend your solution so `test_rate_limit_enforced` and `test_rate_limit_is_per_tenant` pass. The rate limiter must chain on the tenant ID returned by the tenant check — it should not re-parse the header.

**Task 3c: Extend with Audit Logging**

Add the audit logging gatekeeper. This requires careful placement: the gatekeeper must fire even when tenant checking or rate limiting raises an HTTPException.

Write one additional test demonstrating that the audit log captured a rejected request:

```python
def test_audit_log_fires_for_rejected_request():
    """Audit log must record even when tenant check rejects with 403."""
    # Hint: Use a shared list as a simple log sink for testing
    # Your test here
```

---

# Level 3 Solution

## Solution 3: The Multi-Tenant Gatekeeper Chain

```python
# solution.py
from fastapi import FastAPI, APIRouter, Depends, HTTPException, Header, Request
from fastapi.responses import JSONResponse
from typing import Annotated
import time
import structlog

app = FastAPI()
logger = structlog.get_logger()

APPROVED_TENANTS = {"tenant-alpha", "tenant-beta", "tenant-gamma"}
_rate_store: dict[str, list[float]] = {}
RATE_LIMIT = 5
RATE_WINDOW = 10.0


# --- Audit logging gatekeeper (no tenant dependency — reads header directly) ---

audit_log_sink: list[dict] = []   # Simple in-memory sink for testing


async def audit_log_request(request: Request) -> None:
    """Runs for every request. Does not depend on tenant check — fires even on rejection."""
    event = {
        "path": request.url.path,
        "method": request.method,
        "tenant_id": request.headers.get("x-tenant-id", "MISSING"),
        "timestamp": time.time(),
    }
    audit_log_sink.append(event)
    logger.info("api_request", **event)


# --- Tenant identification (returns tenant_id for chaining) ---

def get_tenant_id(x_tenant_id: str = Header(default="")) -> str:
    if not x_tenant_id:
        raise HTTPException(status_code=400, detail="X-Tenant-ID header is required")
    if x_tenant_id not in APPROVED_TENANTS:
        raise HTTPException(status_code=403, detail=f"Unknown tenant: {x_tenant_id}")
    return x_tenant_id


TenantDep = Annotated[str, Depends(get_tenant_id)]


# --- Rate limiter (chains on TenantDep — reuses validated tenant_id) ---

def check_rate_limit(tenant_id: TenantDep) -> None:
    now = time.time()
    window_start = now - RATE_WINDOW
    timestamps = [t for t in _rate_store.get(tenant_id, []) if t > window_start]

    if len(timestamps) >= RATE_LIMIT:
        raise HTTPException(
            status_code=429,
            detail=f"Rate limit: max {RATE_LIMIT} requests per {int(RATE_WINDOW)}s"
        )

    timestamps.append(now)
    _rate_store[tenant_id] = timestamps


# --- Router with ordered dependencies ---

router = APIRouter(
    prefix="/api/v1",
    dependencies=[
        Depends(audit_log_request),  # 1st: always fires, even if next steps reject
        Depends(get_tenant_id),      # 2nd: rejects unknown tenants
        Depends(check_rate_limit),   # 3rd: rejects over-limit tenants
    ]
)


@router.get("/data")
async def get_data():
    return {"data": "tenant-specific payload"}


app.include_router(router)
client_instance = TestClient(app)
```

**Why this satisfies every constraint:**

The `dependencies=` list on the router executes in order. `audit_log_request` runs first — before any `HTTPException` can be raised by the tenant check or rate limiter. Even a 403 or 429 response still triggers the audit log. `check_rate_limit` chains on `TenantDep`, so the tenant ID is resolved once (per-request cache) and passed to the rate limiter automatically — no redundant header parsing. No endpoint has more than 0 parameters because all cross-cutting concerns live at the router level.

**Task 3c: Audit Logging Test**

```python
def test_audit_log_fires_for_rejected_request():
    initial_count = len(audit_log_sink)
    client_instance.get("/api/v1/data", headers={"X-Tenant-ID": "tenant-evil"})
    # At least one new entry was added, despite the 403 rejection
    assert len(audit_log_sink) > initial_count
    last_event = audit_log_sink[-1]
    assert last_event["tenant_id"] == "tenant-evil"
    assert last_event["path"] == "/api/v1/data"
```

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## Exercise 4: When Depends() Becomes the Enemy

`Depends()` is presented as the solution to copy-paste logic. But there are scenarios where using it is actively harmful compared to a simpler alternative.

Describe **two** concrete scenarios where `Depends()` is the wrong tool. For each:

1. Show a small code snippet of the problematic `Depends()` usage
2. Explain what concretely goes wrong (correctness, performance, or maintainability)
3. Show the correct alternative

**Constraint:** You may not use "just call the function directly" as a scenario unless you explain a specific case where FastAPI's caching behavior creates an actual correctness bug — not just a stylistic preference.

---

# Level 4 Solution

## Solution 4: When Depends() Becomes the Enemy

### Scenario A: The Dependency That Must Not Be Cached

```python
# PROBLEMATIC: Using Depends() for a function that must produce unique values
import random

def generate_idempotency_token() -> str:
    return str(random.randint(10000, 99999))

TokenDep = Annotated[str, Depends(generate_idempotency_token)]

@app.post("/payments")
async def create_payment(
    token_for_idempotency: TokenDep,
    token_for_audit: TokenDep,      # Expects a DIFFERENT token
):
    # Both tokens are identical because FastAPI's per-request cache
    # returned the same result for both Depends(generate_idempotency_token)
    return {"idempotency": token_for_idempotency, "audit": token_for_audit}
    # Returns: {"idempotency": "42137", "audit": "42137"}
    # Expected: {"idempotency": "42137", "audit": "87654"}
```

**What goes wrong:** FastAPI's per-request dependency cache (§4.2) sees that `generate_idempotency_token` has already been resolved and reuses the result for the second parameter. When the design requires that each invocation produces a unique value, caching breaks correctness silently — no error is raised, just wrong behavior.

**The correct alternative:**

```python
@app.post("/payments")
async def create_payment():
    token_for_idempotency = generate_idempotency_token()    # Direct call
    token_for_audit = generate_idempotency_token()          # Guaranteed fresh
    return {"idempotency": token_for_idempotency, "audit": token_for_audit}
```

Or, if `Depends()` is required for other reasons: `Depends(generate_idempotency_token, use_cache=False)` on the second parameter.

---

### Scenario B: The Single-Use Dependency Nobody Reuses

```python
# PROBLEMATIC: Wrapping logic used by exactly one endpoint in a dependency
async def get_task_statistics(limit: int = Query(default=10)) -> dict:
    """Computes stats. Used ONLY by GET /stats — nowhere else."""
    all_tasks = list(tasks_db.values())
    completed = [t for t in all_tasks if t["completed"]]
    return {"total": len(all_tasks), "completed": len(completed), "sample": all_tasks[:limit]}

StatsDep = Annotated[dict, Depends(get_task_statistics)]

@app.get("/stats")
async def get_stats(stats: StatsDep):
    return stats
```

**What goes wrong:** `Depends()` exists to *share* logic across multiple endpoints. A dependency used by exactly one endpoint adds indirection with no reuse benefit: the logic is split across two functions (making a simple endpoint harder to trace), the type alias `StatsDep` suggests reusability that doesn't exist, and a future developer may incorrectly apply `StatsDep` to another endpoint expecting generic pagination — but it actually computes task statistics, creating a subtle correctness bug.

**The correct alternative:**

```python
@app.get("/stats")
async def get_stats(limit: int = Query(default=10)):
    """Logic lives here. It's only used here."""
    all_tasks = list(tasks_db.values())
    completed = [t for t in all_tasks if t["completed"]]
    return {"total": len(all_tasks), "completed": len(completed), "sample": all_tasks[:limit]}
```

**The rule:** Create a dependency when the same logic is used by 2+ endpoints, or when it handles a cross-cutting concern (auth, logging, rate limiting). Single-use, endpoint-specific logic belongs in the endpoint itself.

---

# LEVEL 5: THE TEACH BACK

## Exercise 5: Setting the Junior Straight

You are in a code review. A junior developer submitted this pattern used consistently across the entire codebase:

```python
# In every service function:
from fastapi import HTTPException

def get_user(user_id: int) -> dict:
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")
    return users_db[user_id]

def deduct_credits(user_id: int, amount: int) -> dict:
    user = get_user(user_id)       # Can raise HTTPException
    if user["credits"] < amount:
        raise HTTPException(status_code=400, detail="Insufficient credits")
    users_db[user_id]["credits"] -= amount
    return users_db[user_id]
```

They argue:

> "HTTPException is the cleanest approach — it's FastAPI's own error type, it works directly in the service layer without any extra translation step, and I don't need a separate exception class per error type. My endpoint tests all pass. It reduces boilerplate and it works perfectly."

Write a code review response that:
1. Honestly acknowledges what is correct in their argument
2. Demonstrates two concrete failure modes with working code
3. Proposes the correct alternative
4. Is honest about the additional work the correct approach requires

Do not be dismissive. Produce working code for each failure mode.

---

# Level 5 Solution

## Solution 5: Setting the Junior Straight

**To the junior developer:**

Your instinct to reduce boilerplate is valid, and in a prototype with 3 endpoints and no background jobs, this pattern works exactly as you expect. In isolated endpoint tests with `TestClient`, it's functionally correct.

But this pattern has two concrete failure modes that appear the moment the codebase grows beyond a single API entry point.

---

**Failure Mode 1: The service layer becomes a web prisoner**

`HTTPException` is an HTTP concept. Any function that imports and raises it can only be used inside a FastAPI request context. Here is what breaks as soon as you add a background job — which every real project eventually needs:

```python
# celery_tasks.py — a nightly credit expiry job
from service import deduct_credits   # Your function

@celery.task
def expire_user_credits(user_id: int):
    """Runs outside any HTTP request — no client, no response."""
    deduct_credits(user_id, amount=10)
    # If user doesn't exist:
    # → raises HTTPException(status_code=404, detail="User not found")
    # Celery has no concept of HTTP status codes.
    # The task fails with an unexpected exception type.
    # Your error monitoring shows "HTTPException: 404" in background job logs.
    # This is misleading and confusing — there is no HTTP request happening here.
```

```python
# cli.py — a data migration script
from service import get_user

for user_id in [1, 2, 99, 3]:      # 99 was deleted
    user = get_user(user_id)        # Raises HTTPException when user_id=99
    # Script crashes with HTTPException — which has no meaningful
    # representation in a CLI context. The remaining users are never migrated.
```

A `UserNotFoundError` makes sense in all contexts and can be handled appropriately by each caller. An `HTTPException` makes sense only to an HTTP response handler.

---

**Failure Mode 2: The custom exception handler silently stops working**

Suppose you implement the lecture's recommended pattern — a global exception handler for consistent error shapes:

```python
@app.exception_handler(UserNotFoundError)
async def user_not_found_handler(request: Request, exc: UserNotFoundError):
    return JSONResponse(
        status_code=404,
        content={
            "error": "not_found",
            "detail": str(exc),
            "user_id": exc.user_id,   # Structured context for the client
        }
    )
```

But your service raises `HTTPException` directly. The custom handler **never fires** — `HTTPException` bypasses it entirely and hits FastAPI's built-in handler:

```python
# Client receives FastAPI's default:
{"detail": "User not found"}
# Not your structured format:
{"error": "not_found", "detail": "User not found", "user_id": 99}
```

The error contract you promised clients is broken, and there is no warning — the code runs, just with the wrong shape.

---

**The correct alternative — and the honest cost:**

```python
# exceptions.py — pure Python, no FastAPI dependency
class UserNotFoundError(Exception):
    def __init__(self, user_id: int):
        self.user_id = user_id
        super().__init__(f"User {user_id} not found")

class InsufficientCreditsError(Exception):
    def __init__(self, user_id: int, required: int, available: int):
        self.user_id = user_id
        super().__init__(f"User {user_id} needs {required} credits, has {available}")
```

```python
# service.py — no FastAPI import at all
from exceptions import UserNotFoundError, InsufficientCreditsError

def get_user(user_id: int) -> dict:
    if user_id not in users_db:
        raise UserNotFoundError(user_id)     # Works in CLI, Celery, tests, FastAPI
    return users_db[user_id]

def deduct_credits(user_id: int, amount: int) -> dict:
    user = get_user(user_id)
    if user["credits"] < amount:
        raise InsufficientCreditsError(user_id, amount, user["credits"])
    users_db[user_id]["credits"] -= amount
    return users_db[user_id]
```

```python
# main.py — translation layer (the only place that knows about HTTP)
@app.exception_handler(UserNotFoundError)
async def handle_user_not_found(request: Request, exc: UserNotFoundError):
    return JSONResponse(status_code=404, content={
        "error": "not_found", "user_id": exc.user_id, "detail": str(exc)
    })
```

**The honest cost:** You write one exception class per domain error type and one handler per exception class. For a three-endpoint prototype, this is real overhead for small benefit.

The justification becomes clear at scale: the service functions become usable from CLI scripts, Celery tasks, gRPC adapters, and test fixtures without mocking HTTP machinery. The error response format is centrally controlled and guaranteed consistent. When you add a new consumer of the service, it receives structured exceptions it can translate to its own context. The extra code is the price of decoupling — and decoupling is what makes the codebase survivable as it grows.