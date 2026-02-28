# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "Testing FastAPI endpoints is just testing Python functions. I call the endpoint with the right input, check the output, and I'm done. The `TestClient` is a convenience wrapper equivalent to calling the function directly. And if my coverage is high, my API is well-tested."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---|---|
| 1 | `await create_task(task_input)` tests the same thing as `client.post("/tasks", json={...})` | Direct calls bypass routing, Pydantic validation, serialization, middleware, and exception handlers — the full HTTP contract goes untested |
| 2 | A module-level `client = TestClient(app)` is stateless — it's just an HTTP client object | The `app` it wraps has module-level state that persists across tests; a shared client exposes this shared mutable state |
| 3 | `app.dependency_overrides` auto-resets between tests | Overrides persist on the `app` dict until you explicitly call `.clear()` — forgetting this causes override bleed into subsequent tests |
| 4 | `TestClient(app)` and `with TestClient(app) as client` behave identically | Without the context manager, lifespan `startup`/`shutdown` events never fire and `app.state` remains uninitialized |
| 5 | Background tasks are async — they run AFTER `client.post()` returns | TestClient executes background tasks **synchronously before** returning the response object |
| 6 | `json={"key": "val"}` and `data={"key": "val"}` both "send the payload" to the server | They send fundamentally different `Content-Type` headers; an endpoint that expects one will reject the other with 422 |
| 7 | `asyncio_mode = "strict"` is the safer, more explicit mode — use it by default | In `strict` mode, `async def test_*` functions without `@pytest.mark.asyncio` are silently **skipped**, not run |
| 8 | 94% test coverage means 94% of bugs are caught | Coverage measures lines *executed* — a test that never asserts achieves 100% coverage while testing nothing |

## The Main Purpose (Essence)

**Core Insight:** API testing is not about testing Python functions. It is about testing the **HTTP contract** — the binding agreement between your API and every client that talks to it. The contract includes: which status code is returned for which input, what the response body's shape is, which errors are raised and in what format, what headers are set, and how the full request pipeline behaves end-to-end. None of this is visible when calling an endpoint function directly.

**The Problems Each Pattern Solves:**

| Problem | Naive Approach | Correct Approach |
|---------|---------------|-----------------|
| Test a 404 | Catch `HTTPException` from function call | `assert response.status_code == 404` through `TestClient` |
| Test Pydantic validation | Construct a valid model object | Send invalid JSON → verify 422 body's `loc` field |
| Test middleware behavior | Impossible without the HTTP pipeline | `assert "X-Request-ID" in response.headers` |
| Isolate tests from shared state | `dict.clear()` (fragile, misses counters) | `dependency_overrides` with a fresh object per test |
| Test auth enforcement | Pass a user object directly to function | Override `get_current_user`, test the full route |

**The Key Distinctions:**
- `dependency_overrides` operates at the **framework level** — FastAPI intercepts the entire dependency tree, including nested `Depends()` chains
- `TestClient(app)` ≠ `with TestClient(app)` — the lifespan boundary is critical whenever `app.state` is populated during startup
- Knowing **when** to use `AsyncClient` vs `TestClient` is as important as knowing **how** to use either

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "Module-level client is fine" | Test 2 starts with a clean state | Test 2 sees data from Test 1 | App's in-memory storage persists on the module |
| "dependency_overrides auto-clears" | Test 3 uses the real dependency | Test 3 uses the last override from Test 2 | `dependency_overrides` is a plain dict that lives on `app` |
| "`TestClient(app)` triggers startup" | `app.state.config` is populated and readable | `app.state` has no `config` attribute → `AttributeError` → 500 | Lifespan only runs via `__enter__` (the `with` form) |
| "Background tasks run after the response" | `client.post()` returns instantly | `client.post()` blocks until the background task finishes | TestClient runs background tasks synchronously before returning |
| "`json=` and `data=` both send the payload" | Form endpoint accepts the JSON payload | 422 — form parser cannot read `application/json` body | Different `Content-Type` headers; different parsers |
| "`strict` mode = explicit, not skipping" | Tests run, just require decoration | Undecorated `async def test_*` are silently skipped | `strict` mode does not auto-wrap unmarked async tests |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Ghost in the Counter

```python
# items_app.py
from fastapi import FastAPI
from pydantic import BaseModel, Field
from fastapi.testclient import TestClient

app = FastAPI()

items_db: dict[int, dict] = {}
_counter: int = 0

class ItemCreate(BaseModel):
    name: str = Field(min_length=1)

@app.post("/items", status_code=201)
async def create_item(item: ItemCreate):
    global _counter
    _counter += 1
    items_db[_counter] = {"id": _counter, "name": item.name}
    return items_db[_counter]

@app.get("/items")
async def list_items():
    return list(items_db.values())


# ── test_items.py ──────────────────────────────────────────────
client = TestClient(app)   # Module-level: created ONCE for the entire file

def test_first_item_has_id_1():
    response = client.post("/items", json={"name": "Widget"})
    assert response.status_code == 201
    assert response.json()["id"] == 1

def test_list_is_empty_before_any_creates():
    response = client.get("/items")
    assert response.status_code == 200
    assert response.json() == []

def test_second_item_also_has_id_1():
    response = client.post("/items", json={"name": "Gadget"})
    assert response.status_code == 201
    assert response.json()["id"] == 1
```

**Questions:**

1. **Predict** which tests pass and which fail when `pytest` runs them in the order shown. For each failing test, give the exact assertion that fails, the expected value, and the actual value.

2. **Predict** what happens if `pytest` runs the tests in **reverse order** (`test_second_item_also_has_id_1` → `test_list_is_empty_before_any_creates` → `test_first_item_has_id_1`). Does the set of passing tests change? Which tests now pass?

3. A junior developer proposes this fix: add `items_db.clear()` at the top of each test function. **Without running the code**, explain precisely why this fix is incomplete and name the specific test that still fails after the fix is applied.

---

## Exercise 1.2: The Invisible Setup

```python
# config_app.py
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request

class AppConfig:
    def __init__(self, max_connections: int, debug: bool):
        self.max_connections = max_connections
        self.debug = debug

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.config = AppConfig(max_connections=50, debug=False)
    yield
    app.state.config = None

app = FastAPI(lifespan=lifespan)

@app.get("/config")
async def get_config(request: Request):
    # Defensive: uses getattr with a fallback
    config = getattr(request.app.state, "config", None)
    if config is None:
        return {"max_connections": 0, "debug": None, "initialized": False}
    return {
        "max_connections": config.max_connections,
        "debug": config.debug,
        "initialized": True,
    }

@app.get("/config/debug-mode")
async def get_debug_mode(request: Request):
    # Non-defensive: assumes lifespan has already run
    return {"debug": request.app.state.config.debug}


# ── test_config.py ──────────────────────────────────────────────
from fastapi.testclient import TestClient

def test_config_endpoint():
    client = TestClient(app)          # No context manager
    response = client.get("/config")
    assert response.status_code == 200
    data = response.json()
    assert data["max_connections"] == 50   # Will this pass?
    assert data["initialized"] is True     # Will this pass?

def test_debug_mode_endpoint():
    client = TestClient(app)          # No context manager
    response = client.get("/config/debug-mode")
    assert response.status_code == 200    # Will this pass?
    assert response.json()["debug"] is False
```

**Questions:**

1. **Predict** the full outcome of each test. State whether it passes or fails, which assertion fails, and what the actual response body or error looks like.

2. **Explain** the root cause. When does the code inside `lifespan` execute? What specific action triggers it?

3. Both tests have the **same root cause**, but they produce **different symptoms** — one fails with a wrong value, the other fails with a 500. Explain why the same uninitialized state produces two different failure modes. Point to the specific difference in endpoint code that determines which failure occurs.

---

## Exercise 1.3: The Background Task Alibi

```python
# processor_app.py
from fastapi import FastAPI, BackgroundTasks
from fastapi.testclient import TestClient
import time

app = FastAPI()
audit_log: list[dict] = []

def write_audit_entry(action: str, item_id: int) -> None:
    """Simulates a slow audit write (300ms) to an external logging service."""
    time.sleep(0.3)
    audit_log.append({"action": action, "item_id": item_id})

@app.post("/items/{item_id}/process", status_code=202)
async def process_item(item_id: int, background_tasks: BackgroundTasks):
    background_tasks.add_task(write_audit_entry, "processed", item_id)
    return {"status": "accepted", "item_id": item_id}


# ── test_processor.py ──────────────────────────────────────────
import time

client = TestClient(app)

def test_response_is_immediate():
    """
    Developer believes: background tasks are async — they run AFTER
    the response is sent. So client.post() should return in under 50ms.
    """
    start = time.perf_counter()
    response = client.post("/items/7/process")
    elapsed_ms = (time.perf_counter() - start) * 1000

    assert response.status_code == 202
    assert elapsed_ms < 50, f"Expected <50ms response, got {elapsed_ms:.1f}ms"

def test_audit_log_is_empty_immediately_after_response():
    """
    Developer believes: the task hasn't run yet when client.post() returns,
    so the audit_log should still be empty right after the call.
    """
    audit_log.clear()

    response = client.post("/items/99/process")
    assert response.status_code == 202

    # Expect: task hasn't run yet
    assert len(audit_log) == 0, f"Expected empty log, found: {audit_log}"
```

**Questions:**

1. **Predict** the exact outcome of each test. Which assertions fail? Give the actual values.

2. **Explain** the timing difference between TestClient and production. In production, when does `write_audit_entry` execute relative to when the client receives the HTTP response? In TestClient, when does it execute? Why does this difference exist?

3. **Rewrite** `test_audit_log_is_empty_immediately_after_response` so it tests a meaningful property of the endpoint: that `write_audit_entry` is called with the correct arguments. Use `unittest.mock.patch`. Your rewritten test must have no timing dependency and must not be flaky.

---

## Exercise 1.4: The Content-Type Impersonator

```python
# auth_app.py
from fastapi import FastAPI, Form
from pydantic import BaseModel
from fastapi.testclient import TestClient

app = FastAPI()

class LoginPayload(BaseModel):
    username: str
    password: str

@app.post("/auth/json")
async def login_json(credentials: LoginPayload):
    """Expects application/json body → Pydantic model."""
    if credentials.username == "alice" and credentials.password == "secret":
        return {"token": "jwt-abc"}
    return {"error": "invalid"}

@app.post("/auth/form")
async def login_form(
    username: str = Form(...),
    password: str = Form(...),
):
    """Expects application/x-www-form-urlencoded → Form fields."""
    if username == "alice" and password == "secret":
        return {"token": "jwt-abc"}
    return {"error": "invalid"}

client = TestClient(app)

# Developer mixes up json= and data=:
def test_json_endpoint_with_data_kwarg():
    response = client.post(
        "/auth/json",
        data={"username": "alice", "password": "secret"}  # ← uses data=
    )
    print(f"Status: {response.status_code}")

def test_form_endpoint_with_json_kwarg():
    response = client.post(
        "/auth/form",
        json={"username": "alice", "password": "secret"}  # ← uses json=
    )
    print(f"Status: {response.status_code}")
```

**Questions:**

1. **Predict** the HTTP status code for each test. Pick one value per test — do not write "it depends." Justify each prediction in one sentence.

2. For each test that returns 422: identify the exact `loc` value in the error detail that FastAPI reports. Does it point to `["body", "username"]`, `["body", "password"]`, `["form", "username"]`, `["body"]`, or something else? Explain why.

3. **Explain the deep mechanism.** When FastAPI receives an `application/x-www-form-urlencoded` request body on an endpoint that declares `credentials: LoginPayload`, what does the framework actually attempt to do with the body? Trace the parse attempt step-by-step until it fails.

---

# Level 1 Solutions

## Solution 1.1: The Ghost in the Counter

**1. In-order prediction:**

| Test | Result | Failing Assertion | Expected | Actual |
|------|--------|-------------------|----------|--------|
| `test_first_item_has_id_1` | ✅ PASSES | — | — | — |
| `test_list_is_empty_before_any_creates` | ❌ FAILS | `assert response.json() == []` | `[]` | `[{"id": 1, "name": "Widget"}]` |
| `test_second_item_also_has_id_1` | ❌ FAILS | `assert response.json()["id"] == 1` | `1` | `2` |

`test_list_is_empty_before_any_creates` runs second. By then, `items_db` already contains `{1: {"id": 1, "name": "Widget"}}` from test 1. The GET returns that data, not `[]`.

`test_second_item_also_has_id_1` runs third. `_counter` is already `1`. This test increments it to `2`, so the new item gets `id=2`, not `1`.

**2. Reverse-order prediction:**

| Test | Result |
|------|--------|
| `test_second_item_also_has_id_1` | ✅ PASSES — first item ever created, gets `id=1` |
| `test_list_is_empty_before_any_creates` | ❌ FAILS — list has 1 item |
| `test_first_item_has_id_1` | ❌ FAILS — second item gets `id=2` |

The set of passing tests changes: `test_second_item_also_has_id_1` now passes instead of `test_first_item_has_id_1`. This is the hallmark of order-dependent tests — which test "wins" depends on who runs first.

**3. Why `items_db.clear()` is incomplete:**

`items_db.clear()` empties the dictionary, but `_counter` is a module-level **integer**. Integers are immutable in Python — you cannot "clear" an integer through a mutable container operation. `_counter` remains at whatever value previous tests left it.

After test 1 runs (`_counter = 1`) and `items_db.clear()` is called at the start of test 2, `_counter` is still `1`. When test 2's `create_item` runs, it increments to `2` and assigns `id=2`. The assertion `assert response.json()["id"] == 1` still fails. The fix requires resetting both the dictionary AND the counter, which is exactly why a `dependency_overrides`-injected object that encapsulates both as **instance variables** is the correct solution.

---

## Solution 1.2: The Invisible Setup

**1. Test outcomes:**

`test_config_endpoint`:
- Status: **200** (does not crash — the endpoint uses `getattr(..., None)`)
- Response body: `{"max_connections": 0, "debug": None, "initialized": False}`
- Failing assertion: `assert data["max_connections"] == 50` — actual is `0`
- The `if config is None` guard returns the fallback. Test fails on the value check, not the status.

`test_debug_mode_endpoint`:
- Status: **500 Internal Server Error**
- Error: FastAPI catches `AttributeError: 'State' object has no attribute 'config'` and returns 500
- First failing assertion: `assert response.status_code == 200` — actual is `500`

**2. Lifespan trigger:**

`app.state.config` is set inside the `lifespan` async context manager. This only executes when `TestClient` is used as a context manager — specifically, when its `__enter__` method is called. `TestClient(app)` (without `with`) constructs the client but never calls `__enter__`, so the lifespan generator never starts and `app.state.config` is never assigned.

**3. Same root cause, different symptoms:**

`get_config` uses `getattr(request.app.state, "config", None)`. Starlette's `State` object raises `AttributeError` when you access a nonexistent attribute directly, but `getattr(obj, name, default)` catches that error internally and returns the `default` value (`None`). The `if config is None` check then handles the missing state gracefully, returning a "not initialized" dict with status 200. Wrong data, no crash.

`get_debug_mode` uses `request.app.state.config.debug` — a direct chained attribute access with no fallback. `AttributeError` is raised and propagates as an unhandled exception. FastAPI converts this to a 500 response.

**Same cause — lifespan not triggered — two symptoms — wrong value vs crash — determined by whether the endpoint is defensively coded, not by TestClient.**

---

## Solution 1.3: The Background Task Alibi

**1. Test outcomes:**

`test_response_is_immediate`:
- **Fails.** `elapsed_ms` is approximately **300ms**, not under 50ms. The `time.sleep(0.3)` inside `write_audit_entry` ran synchronously inside `client.post()`.

`test_audit_log_is_empty_immediately_after_response`:
- **Fails.** `audit_log` contains `[{"action": "processed", "item_id": 99}]` — the task already ran before `client.post()` returned.

**2. Production vs TestClient timing:**

In **production** (uvicorn): The endpoint function returns a response object → FastAPI serializes it → the HTTP 202 is sent over the network to the client. The client receives the response. *Only then*, after the response transmission is complete, does uvicorn's event loop begin running the background task.

In **TestClient**: The background task runs **synchronously** as part of Starlette's test transport response cycle. `client.post()` does not return until the task finishes. This is a deliberate design choice: it makes background task testing deterministic without requiring `asyncio.sleep()` or polling.

**3. Rewritten test:**

```python
from unittest.mock import patch

def test_audit_entry_is_written_with_correct_arguments():
    """write_audit_entry must be called with the correct action and item_id."""
    with patch("processor_app.write_audit_entry") as mock_audit:
        response = client.post("/items/99/process")

        assert response.status_code == 202
        # The mock was called exactly once (task was scheduled and ran)
        mock_audit.assert_called_once_with("processed", 99)
```

`patch` replaces `write_audit_entry` with a mock — no sleep, no actual log write. The mock records the call, so we can assert on the exact arguments. No timing dependency, no flakiness.

---

## Solution 1.4: The Content-Type Impersonator

**1. Status code predictions:**

- `test_json_endpoint_with_data_kwarg`: **422** — `data={}` sends `Content-Type: application/x-www-form-urlencoded`. The endpoint expects a JSON body to parse into `LoginPayload`. The body cannot be parsed as JSON.

- `test_form_endpoint_with_json_kwarg`: **422** — `json={}` sends `Content-Type: application/json`. The endpoint declares `Form(...)` parameters, which require `application/x-www-form-urlencoded`. FastAPI finds no form fields in the JSON body.

**2. The `loc` field for each 422:**

`test_json_endpoint_with_data_kwarg`: FastAPI tries to JSON-parse the form-encoded body (`username=alice&password=secret`). `json.loads()` fails immediately. The error's `loc` is `["body"]` with type `"json_invalid"` — the entire body failed to parse, not a specific field.

`test_form_endpoint_with_json_kwarg`: FastAPI looks for form fields `username` and `password` in the form data portion of the request. A JSON body has no form data. Both fields are reported missing: `loc: ["body", "username"]` and `loc: ["body", "password"]`, type `"missing"`.

**3. The deep mechanism:**

When FastAPI receives a request for `/auth/json`:
1. It inspects the endpoint signature: `credentials: LoginPayload` — a Pydantic `BaseModel`
2. FastAPI determines this must come from the JSON body → it calls `await request.json()` to deserialize the body
3. `data={"username": "alice", "password": "secret"}` sends the raw bytes `username=alice&password=secret` with `Content-Type: application/x-www-form-urlencoded`
4. `json.loads(b"username=alice&password=secret")` raises `json.JSONDecodeError` — that string is not valid JSON
5. FastAPI catches the decode error and returns 422 with `loc: ["body"]`

FastAPI does not "try form parsing as a fallback" for Pydantic body parameters. The `Content-Type` header is the contract: the server uses it to select the parser. When the header says form-encoded but the endpoint expects JSON (or vice versa), the wrong parser is applied and the body fails to parse.

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Amnesiac Override

**Context:** A developer wrote three tests for protected endpoints. Each test passes when run alone with `pytest -k test_name`. When run together, one mysteriously fails.

```python
# protected_app.py
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel

app = FastAPI()

class User(BaseModel):
    id: int
    username: str
    role: str

async def get_current_user() -> User:
    """Production: decodes JWT. Raises 401 so tests MUST override this."""
    raise HTTPException(status_code=401, detail="No token provided")

async def require_admin(user: User = Depends(get_current_user)) -> User:
    if user.role != "admin":
        raise HTTPException(status_code=403, detail="Admin access required")
    return user

@app.get("/me")
async def get_profile(user: User = Depends(get_current_user)):
    return {"id": user.id, "username": user.username}

@app.delete("/admin/records/{record_id}", status_code=204)
async def delete_record(record_id: int, _: User = Depends(require_admin)):
    return None


# ── test_protected.py (BUGGY) ─────────────────────────────────
from fastapi.testclient import TestClient

client = TestClient(app)

def test_admin_can_delete_records():
    app.dependency_overrides[get_current_user] = lambda: User(
        id=1, username="superuser", role="admin"
    )
    response = client.delete("/admin/records/42")
    assert response.status_code == 204
    # ← No cleanup of dependency_overrides

def test_regular_user_sees_profile():
    app.dependency_overrides[get_current_user] = lambda: User(
        id=2, username="alice", role="user"
    )
    response = client.get("/me")
    assert response.status_code == 200
    assert response.json()["username"] == "alice"
    # ← No cleanup of dependency_overrides

def test_request_without_token_is_rejected():
    # Developer intends: no override set → real dependency → 401
    response = client.get("/me")
    assert response.status_code == 401
```

**Task 2a: Identify the Flaw**

Run the tests mentally in the order shown. Which test(s) fail? Give the exact failing assertion and its actual vs expected value.

**Task 2b: Explain the Mechanism**

1. `app.dependency_overrides` is a plain Python `dict` on the `app` object. Draw the state of this dict after each test function body executes, before any cleanup runs.
2. When `test_request_without_token_is_rejected` runs, which function does FastAPI actually invoke when resolving `Depends(get_current_user)`? Describe the exact lookup FastAPI performs.

**Task 2c: Instrument to Prove Diagnosis**

Add exactly **one** `print` statement to `test_request_without_token_is_rejected` that reveals which function is currently registered as the override for `get_current_user`. Show what the output looks like:

```python
def test_request_without_token_is_rejected():
    # One print statement here
    print(f"DEBUG: {???}")
    response = client.get("/me")
    assert response.status_code == 401
```

**Task 2d: Fix the Code**

Rewrite all three tests so that:
- Each test has the exact override state it requires
- Override cleanup is **guaranteed** even if an assertion fails mid-test
- Do NOT convert to pytest fixtures yet — keep the module-level `client`

**Task 2e: Write a Verification Test**

Write a self-contained test that would **FAIL** on the original broken code and **PASS** on your fixed version. The test must not rely on execution order — it must demonstrate the isolation property directly within a single function.

---

## Exercise 2.2: The Leaking Lifespan

**Context:** A developer is testing an API that reads rate-limiting policy from `app.state`. Their tests pass in isolation but fail in CI where the full suite runs sequentially.

```python
# rate_app.py
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request
from pydantic import BaseModel

class RatePolicy(BaseModel):
    limit: int
    window_seconds: int

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.policy = RatePolicy(limit=100, window_seconds=60)
    yield
    del app.state.policy

app = FastAPI(lifespan=lifespan)

@app.get("/policy")
async def get_policy(request: Request):
    return request.app.state.policy.model_dump()

@app.patch("/policy")
async def update_policy(new_policy: RatePolicy, request: Request):
    request.app.state.policy = new_policy
    return request.app.state.policy.model_dump()


# ── test_rate.py (BUGGY) ──────────────────────────────────────
from fastapi.testclient import TestClient

def test_default_policy():
    client = TestClient(app)               # ← No context manager
    response = client.get("/policy")
    assert response.status_code == 200
    assert response.json()["limit"] == 100

def test_update_policy():
    client = TestClient(app)               # ← No context manager
    update_resp = client.patch(
        "/policy",
        json={"limit": 10, "window_seconds": 30}
    )
    assert update_resp.status_code == 200
    get_resp = client.get("/policy")
    assert get_resp.json()["limit"] == 10

def test_policy_resets_between_tests():
    # Developer assumes: each new TestClient(app) gives a fresh app state
    client = TestClient(app)               # ← No context manager
    response = client.get("/policy")
    assert response.status_code == 200
    assert response.json()["limit"] == 100  # Will this be 100 or 10?
```

**Task 2a: Identify the Flaw**

For each test, predict the exact outcome when they run in the order shown. Consider what `TestClient(app)` does vs `with TestClient(app)`, and what happens to `app.state` across tests on the same module-level `app` object.

**Task 2b: Explain the Mechanism**

1. When `TestClient(app)` is instantiated without `with`, does any part of the `lifespan` generator execute? Why or why not?
2. `test_update_policy` calls `PATCH /policy` which assigns to `request.app.state.policy`. Does this require lifespan to have previously run? What is the difference between **writing to** and **reading from** `app.state`?
3. `app` is a module-level singleton. When `test_update_policy` sets `app.state.policy = RatePolicy(limit=10, ...)`, and then `test_policy_resets_between_tests` instantiates `TestClient(app)` — is this a fresh `app`? Does `app.state` reset?

**Task 2c: Instrument to Prove Diagnosis**

Add print statements that expose `app.state` contents at the start of `test_policy_resets_between_tests`. Show the output given the described test order.

```python
def test_policy_resets_between_tests():
    print(f"DEBUG: has 'policy' attr: {hasattr(app.state, 'policy')}")
    if hasattr(app.state, "policy"):
        print(f"DEBUG: policy.limit = {app.state.policy.limit}")
    client = TestClient(app)
    response = client.get("/policy")
    assert response.json()["limit"] == 100
```

**Task 2d: Fix the Code**

Rewrite all three tests to:
1. Correctly trigger lifespan events in each test
2. Ensure each test starts with the default policy (`limit=100`)
3. Prevent `test_update_policy`'s mutation from bleeding into subsequent tests

**Task 2e: Write a Verification Test**

Write a single test that proves all three properties:
- Startup ran (default policy is populated)
- An update within the test is visible within the same test
- Shutdown ran (`app.state.policy` is removed after the `with` block exits)

This test must **FAIL** if `TestClient(app)` is used without the context manager.

---

# Level 2 Solutions

## Solution 2.1: The Amnesiac Override

**Task 2a: Identify the Flaw**

`test_request_without_token_is_rejected` fails.

Exact failure:
```
AssertionError: assert 200 == 401
```

After `test_regular_user_sees_profile` completes (without cleanup), `app.dependency_overrides` is:
```python
{get_current_user: <lambda: User(id=2, username="alice", role="user")>}
```
When `test_request_without_token_is_rejected` runs, FastAPI finds this override and returns alice's profile with 200 instead of calling the real `get_current_user` that raises 401.

**Task 2b: Explain the Mechanism**

State of `app.dependency_overrides` after each test:

| Point in time | Dict state |
|---|---|
| Before any tests | `{}` |
| After `test_admin_can_delete_records` | `{get_current_user: <admin lambda>}` |
| After `test_regular_user_sees_profile` | `{get_current_user: <alice lambda>}` (overwritten) |
| When `test_request_without_token_is_rejected` starts | `{get_current_user: <alice lambda>}` — still there! |

FastAPI resolves `Depends(get_current_user)` by:
1. Checking `app.dependency_overrides` for `get_current_user` as a key
2. If found → call the override function
3. If not found → call the original `get_current_user`

Since the dict was never cleared, step 2 fires. The real 401-raising function never runs.

**Task 2c: Instrument to Prove Diagnosis**

```python
def test_request_without_token_is_rejected():
    print(f"DEBUG: {app.dependency_overrides.get(get_current_user, 'NOT OVERRIDDEN')}")
    response = client.get("/me")
    assert response.status_code == 401
```

Output when run after the other two tests:
```
DEBUG: <function test_regular_user_sees_profile.<locals>.<lambda> at 0x7f...>
```

The print shows the lambda from test 2 is still registered. `NOT OVERRIDDEN` would only appear if the dict were properly cleared.

**Task 2d: Fix the Code**

```python
def test_admin_can_delete_records():
    app.dependency_overrides[get_current_user] = lambda: User(
        id=1, username="superuser", role="admin"
    )
    try:
        response = client.delete("/admin/records/42")
        assert response.status_code == 204
    finally:
        app.dependency_overrides.clear()   # Guaranteed even if assertion fails

def test_regular_user_sees_profile():
    app.dependency_overrides[get_current_user] = lambda: User(
        id=2, username="alice", role="user"
    )
    try:
        response = client.get("/me")
        assert response.status_code == 200
        assert response.json()["username"] == "alice"
    finally:
        app.dependency_overrides.clear()

def test_request_without_token_is_rejected():
    try:
        response = client.get("/me")
        assert response.status_code == 401
    finally:
        app.dependency_overrides.clear()   # Consistent cleanup in every test
```

**Task 2e: Verification Test**

```python
def test_override_isolation_is_guaranteed():
    """
    Proves cleanup: sets an override, cleans up, then verifies the real
    dependency is active. FAILS on original code. PASSES after fix.
    """
    # Step 1: Set an override temporarily
    app.dependency_overrides[get_current_user] = lambda: User(
        id=99, username="ghost", role="user"
    )
    try:
        r1 = client.get("/me")
        assert r1.status_code == 200   # Override active → ghost user
        assert r1.json()["username"] == "ghost"
    finally:
        app.dependency_overrides.clear()

    # Step 2: After cleanup, the real dependency must be active again
    r2 = client.get("/me")
    assert r2.status_code == 401      # Real function → 401
```

This test self-contains the "before cleanup" and "after cleanup" states in one function, independent of any other test's execution order.

---

## Solution 2.2: The Leaking Lifespan

**Task 2a: Identify the Flaw**

`test_default_policy`:
- `TestClient(app)` → lifespan never runs → `request.app.state.policy` → `AttributeError` → **500 Internal Server Error**
- Fails: `assert response.status_code == 200`

`test_update_policy`:
- `TestClient(app)` again → lifespan still never ran
- PATCH `/policy`: the handler does `request.app.state.policy = new_policy` — this **writes** a new attribute, which does not require lifespan. **Succeeds.**
- GET `/policy`: now `app.state.policy` exists (written by the PATCH). Returns `{"limit": 10, ...}`.
- `assert get_resp.json()["limit"] == 10` → **PASSES** (but for the wrong reason: the test works despite the bug)

`test_policy_resets_between_tests`:
- `TestClient(app)` — same `app` singleton. `app.state.policy` is still `RatePolicy(limit=10, ...)` from the previous test's PATCH.
- GET returns `{"limit": 10}`.
- Fails: `assert response.json()["limit"] == 100` — actual is `10`.

**Task 2b: Explain the Mechanism**

1. No code from `lifespan` executes. `TestClient(app)` constructs the sync client but `__enter__` is never called, so the `asynccontextmanager` generator never starts. The code before `yield` in `lifespan` is unreachable.

2. **Writing vs reading from `app.state`**: Reading a nonexistent attribute (`request.app.state.policy`) raises `AttributeError`. Writing a new attribute (`request.app.state.policy = value`) uses Python's normal `setattr` path — it creates the attribute if it doesn't exist. Writing succeeds whether or not lifespan ran; reading only succeeds if the attribute was previously set.

3. `app` is a module-level Python object. All three `TestClient(app)` calls wrap the identical `app` instance. `app.state` is an attribute of that instance — not a per-`TestClient` copy. When test 2 sets `app.state.policy = RatePolicy(limit=10, ...)`, that mutation persists on the shared singleton. Test 3's `TestClient(app)` creates a new wrapper object, but the underlying `app` is unchanged. No reset happens.

**Task 2c: Instrument Output**

Given `test_update_policy` runs before `test_policy_resets_between_tests`:
```
DEBUG: has 'policy' attr: True
DEBUG: policy.limit = 10
```

The attribute exists (set by test 2's PATCH request), but its value is `10`, not `100`. The assertion fails.

**Task 2d: Fix the Code**

```python
def test_default_policy():
    with TestClient(app) as client:      # ← Context manager: triggers lifespan startup
        response = client.get("/policy")
        assert response.status_code == 200
        assert response.json()["limit"] == 100

def test_update_policy():
    with TestClient(app) as client:      # ← Fresh startup; shutdown cleans up after
        update_resp = client.patch(
            "/policy",
            json={"limit": 10, "window_seconds": 30}
        )
        assert update_resp.status_code == 200
        get_resp = client.get("/policy")
        assert get_resp.json()["limit"] == 10

def test_policy_resets_between_tests():
    with TestClient(app) as client:      # ← Each with-block: startup → default policy
        response = client.get("/policy")
        assert response.status_code == 200
        assert response.json()["limit"] == 100   # Passes now
```

Each `with` block triggers the full lifespan: startup sets the default policy, shutdown deletes it. State from one test block cannot reach the next.

**Task 2e: Verification Test**

```python
def test_lifespan_fully_executes_and_cleans_up():
    """
    Proves startup ran, updates are visible, and shutdown cleaned up.
    FAILS if TestClient(app) is used without the context manager.
    """
    # Precondition: policy must not exist before we enter the context
    assert not hasattr(app.state, "policy"), \
        "policy should not exist before context manager entry"

    with TestClient(app) as client:
        # Startup ran: default policy is present
        get_resp = client.get("/policy")
        assert get_resp.status_code == 200
        assert get_resp.json()["limit"] == 100      # Startup ran ✓

        # Update is visible within the same context
        client.patch("/policy", json={"limit": 5, "window_seconds": 10})
        updated = client.get("/policy")
        assert updated.json()["limit"] == 5         # Mutation applied ✓

    # Shutdown ran: policy attribute is removed
    assert not hasattr(app.state, "policy"), \
        "policy should be deleted after context manager exits (shutdown ran)"
```

---

# LEVEL 2.5: EXTEND EXISTING CODE

**Given: A working note-taking API and a passing baseline test suite.**

```python
# notes_app.py
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, Field

app = FastAPI()

class NoteCreate(BaseModel):
    title: str = Field(min_length=1, max_length=80)
    body: str = Field(min_length=1, max_length=5000)
    pinned: bool = False

class NoteResponse(BaseModel):
    id: int
    title: str
    body: str
    pinned: bool

class NoteStorage:
    def __init__(self):
        self._notes: dict[int, dict] = {}
        self._counter: int = 0

    def create(self, data: dict) -> dict:
        self._counter += 1
        record = {"id": self._counter, **data}
        self._notes[self._counter] = record
        return record

    def get(self, note_id: int) -> dict | None:
        return self._notes.get(note_id)

    def list_all(self) -> list[dict]:
        return list(self._notes.values())

    def delete(self, note_id: int) -> bool:
        if note_id not in self._notes:
            return False
        del self._notes[note_id]
        return True

_storage = NoteStorage()

def get_storage() -> NoteStorage:
    return _storage

@app.post("/notes", status_code=201)
async def create_note(
    note: NoteCreate,
    storage: NoteStorage = Depends(get_storage),
) -> NoteResponse:
    return NoteResponse(**storage.create(note.model_dump()))

@app.get("/notes/{note_id}")
async def get_note(
    note_id: int,
    storage: NoteStorage = Depends(get_storage),
) -> NoteResponse:
    note = storage.get(note_id)
    if note is None:
        raise HTTPException(status_code=404, detail=f"Note {note_id} not found")
    return NoteResponse(**note)

@app.get("/notes")
async def list_notes(
    storage: NoteStorage = Depends(get_storage),
) -> list[NoteResponse]:
    return [NoteResponse(**n) for n in storage.list_all()]

@app.delete("/notes/{note_id}", status_code=204)
async def delete_note(
    note_id: int,
    storage: NoteStorage = Depends(get_storage),
) -> None:
    if not storage.delete(note_id):
        raise HTTPException(status_code=404, detail=f"Note {note_id} not found")
```

```python
# tests/conftest.py  — BASELINE: do not modify this file
import pytest
from fastapi.testclient import TestClient
from notes_app import app, get_storage, NoteStorage

@pytest.fixture
def storage() -> NoteStorage:
    return NoteStorage()

@pytest.fixture
def client(storage: NoteStorage) -> TestClient:
    app.dependency_overrides[get_storage] = lambda: storage
    yield TestClient(app)
    app.dependency_overrides.clear()
```

```python
# tests/test_notes.py  — BASELINE: all tests must still pass after each extension
import pytest
from fastapi.testclient import TestClient

def test_create_note_returns_201(client):
    response = client.post("/notes", json={"title": "Hello", "body": "World"})
    assert response.status_code == 201

def test_get_existing_note(client):
    create_resp = client.post("/notes", json={"title": "My Note", "body": "Content"})
    note_id = create_resp.json()["id"]
    response = client.get(f"/notes/{note_id}")
    assert response.status_code == 200
    assert response.json()["title"] == "My Note"

def test_get_nonexistent_note_returns_404(client):
    response = client.get("/notes/99999")
    assert response.status_code == 404

def test_list_notes(client):
    client.post("/notes", json={"title": "Note 1", "body": "Body 1"})
    client.post("/notes", json={"title": "Note 2", "body": "Body 2"})
    response = client.get("/notes")
    assert response.status_code == 200
    assert len(response.json()) == 2
```

**The baseline tests above must pass after every extension you add.**

---

### Extension 1: Parametrized Validation Boundaries

Add a single `@pytest.mark.parametrize` test function that systematically covers all Pydantic validation boundaries for `NoteCreate`.

**Requirements:**
- Cover `title`: empty string (invalid), exactly 1 char (valid), exactly 80 chars (valid), 81 chars (invalid)
- Cover `body`: empty string (invalid), 1 char (valid), 5000 chars (valid), 5001 chars (invalid)
- Cover `pinned`: absent (should default to `False` in response), `True`, and `False` explicitly — for valid `pinned` cases, assert the response body value, not just the status code
- Use **one** `@pytest.mark.parametrize` decorator with `pytest.param(..., id="human-readable-name")` for every case
- The test function signature must accept `client`, `payload`, `expected_status`, and any additional parameters needed for body assertions

**Constraint:** Do not write a separate test function for each case. Do not test combinations of invalid fields in a single test case — one validation boundary at a time.

---

### Extension 2: Response Shape Contract Guard

Add a test that acts as a shape contract: it verifies that `GET /notes/{id}` returns **exactly** the fields declared in `NoteResponse` — no more, no fewer.

**Requirements:**
- If a developer adds `created_at: datetime` to `NoteResponse`, the test automatically detects and includes that field in its expectation — it must **not** break on intentional model changes
- If the API response contains a field not in `NoteResponse`, the test fails
- If the API response is missing a field that is in `NoteResponse`, the test fails

**Constraint:** Do NOT hardcode the expected set as `{"id", "title", "body", "pinned"}` in your assertion. Derive the expected fields programmatically from `NoteResponse`'s own definition.

---

### Extension 3: Concurrent Delete Race

Using `httpx.AsyncClient` with `ASGITransport`, write an async test that verifies delete is correctly idempotent under concurrent load.

**Requirements:**
1. Create one note
2. Fire exactly **5 concurrent DELETE requests** to the same note ID using `asyncio.gather`
3. Assert exactly **1** response has status 204 and exactly **4** have status 404
4. Confirm the note is truly gone: a subsequent GET returns 404

**Constraints:**
- Do NOT use `TestClient` — must use `httpx.AsyncClient`
- The test function must be `async def`
- Write a **new async fixture** for this test — do not modify the baseline `client` fixture
- The baseline synchronous tests must continue to work after adding this fixture

---

# Level 2.5 Solutions

## Solution: Extension 1

```python
# tests/test_notes.py — Extension 1 (append to existing file)
@pytest.mark.parametrize(
    "payload, expected_status, check_field, expected_value",
    [
        # ── title: invalid ───────────────────────────────────────────────────
        pytest.param(
            {"title": "", "body": "valid body"},
            422, None, None,
            id="title-empty-rejected"
        ),
        pytest.param(
            {"title": "x" * 81, "body": "valid body"},
            422, None, None,
            id="title-81-chars-rejected"
        ),
        # ── title: valid ─────────────────────────────────────────────────────
        pytest.param(
            {"title": "x", "body": "valid body"},
            201, "title", "x",
            id="title-1-char-accepted"
        ),
        pytest.param(
            {"title": "x" * 80, "body": "valid body"},
            201, "title", "x" * 80,
            id="title-80-chars-accepted"
        ),
        # ── body: invalid ────────────────────────────────────────────────────
        pytest.param(
            {"title": "valid title", "body": ""},
            422, None, None,
            id="body-empty-rejected"
        ),
        pytest.param(
            {"title": "valid title", "body": "x" * 5001},
            422, None, None,
            id="body-5001-chars-rejected"
        ),
        # ── body: valid ──────────────────────────────────────────────────────
        pytest.param(
            {"title": "valid title", "body": "x"},
            201, "body", "x",
            id="body-1-char-accepted"
        ),
        pytest.param(
            {"title": "valid title", "body": "x" * 5000},
            201, "body", "x" * 5000,
            id="body-5000-chars-accepted"
        ),
        # ── pinned: default and explicit ─────────────────────────────────────
        pytest.param(
            {"title": "valid title", "body": "valid body"},
            201, "pinned", False,    # omitted → should default to False
            id="pinned-absent-defaults-false"
        ),
        pytest.param(
            {"title": "valid title", "body": "valid body", "pinned": True},
            201, "pinned", True,
            id="pinned-true-accepted"
        ),
        pytest.param(
            {"title": "valid title", "body": "valid body", "pinned": False},
            201, "pinned", False,
            id="pinned-explicit-false-accepted"
        ),
    ],
)
def test_note_create_validation(
    client: TestClient,
    payload: dict,
    expected_status: int,
    check_field: str | None,
    expected_value,
):
    response = client.post("/notes", json=payload)
    assert response.status_code == expected_status

    if check_field is not None:
        assert response.json()[check_field] == expected_value
```

**Why this design works:** The boundary values 1/80 and 1/5000 are the only ones that matter — anything between the boundaries provides no additional information. `pytest.param` with `id=` makes the output readable: `test_note_create_validation[title-empty-rejected]` is immediately understandable in CI output. The `check_field` / `expected_value` pair lets the same function body both check-status-only (for 422 cases) and check-status-plus-value (for 201 cases) without any conditional complexity in the parametrize decorator.

---

## Solution: Extension 2

```python
# tests/test_notes.py — Extension 2
from notes_app import NoteResponse

def test_response_shape_matches_noteresponse_model(client: TestClient):
    """
    Contract guard: GET /notes/{id} response must contain exactly the
    fields declared in NoteResponse — derived from the model, not hardcoded.
    """
    create_resp = client.post(
        "/notes",
        json={"title": "Shape contract test", "body": "Verifying the contract"}
    )
    note_id = create_resp.json()["id"]

    response = client.get(f"/notes/{note_id}")
    assert response.status_code == 200

    # Derive expected fields from the model — no hardcoding
    expected_fields = set(NoteResponse.model_fields.keys())
    actual_fields = set(response.json().keys())

    assert actual_fields == expected_fields, (
        f"Response shape mismatch.\n"
        f"  Expected (NoteResponse fields): {sorted(expected_fields)}\n"
        f"  Actual   (API response fields): {sorted(actual_fields)}\n"
        f"  Extra fields in response:   {actual_fields - expected_fields}\n"
        f"  Missing fields in response: {expected_fields - actual_fields}"
    )
```

**Why `model_fields` works:** `NoteResponse.model_fields` is a Pydantic v2 dict mapping field names to `FieldInfo` objects. `set(NoteResponse.model_fields.keys())` returns `{"id", "title", "body", "pinned"}`. If a developer intentionally adds `created_at: datetime` to the model, `expected_fields` automatically includes `created_at` — the test adapts rather than breaking. If the API response leaks an internal field not in the model, `actual_fields - expected_fields` is non-empty and the assertion fails with a precise diff message.

---

## Solution: Extension 3

```python
# tests/conftest.py — add this fixture; do not modify existing fixtures
import pytest
from httpx import AsyncClient, ASGITransport
from notes_app import app, get_storage, NoteStorage

@pytest.fixture
async def async_client_isolated():
    """Async client with its own isolated NoteStorage. For concurrent tests only."""
    test_storage = NoteStorage()
    app.dependency_overrides[get_storage] = lambda: test_storage

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client

    app.dependency_overrides.clear()
```

```python
# tests/test_notes.py — Extension 3
import asyncio
import pytest
from httpx import AsyncClient

async def test_concurrent_deletes_are_idempotent(
    async_client_isolated: AsyncClient,
):
    """
    5 concurrent DELETEs to the same note: exactly 1 must return 204,
    exactly 4 must return 404.
    """
    client = async_client_isolated

    # Step 1: Create one note
    create_resp = await client.post(
        "/notes",
        json={"title": "Race target", "body": "Delete me concurrently"}
    )
    assert create_resp.status_code == 201
    note_id = create_resp.json()["id"]

    # Step 2: Fire 5 concurrent DELETEs
    responses = await asyncio.gather(*[
        client.delete(f"/notes/{note_id}") for _ in range(5)
    ])

    # Step 3: Exactly 1 success, 4 not-founds
    status_codes = [r.status_code for r in responses]
    assert status_codes.count(204) == 1, (
        f"Expected exactly 1×204, got: {status_codes}"
    )
    assert status_codes.count(404) == 4, (
        f"Expected exactly 4×404, got: {status_codes}"
    )

    # Step 4: Note is truly gone
    verify = await client.get(f"/notes/{note_id}")
    assert verify.status_code == 404
```

**Why `AsyncClient` and not `TestClient` here:** `TestClient` is synchronous. Even if you write five sequential `client.delete()` calls in a loop, they execute one after another — the "race" never happens; the first call always gets 204 and the rest always get 404. `asyncio.gather()` with `AsyncClient` dispatches all five coroutines onto the event loop simultaneously, creating a genuine concurrent scenario that exercises the "already deleted" code path from multiple callers at the same logical instant.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## The Application

You are joining a team. You receive the following FastAPI application with zero tests. Three rules apply:

**Rule 1:** You cannot modify `survey_app.py`.  
**Rule 2:** Each test must be fully independent — running any subset in any order must produce identical results.  
**Rule 3:** The suite must achieve ≥80% line coverage.

```python
# survey_app.py
from fastapi import FastAPI, HTTPException, Depends, BackgroundTasks
from pydantic import BaseModel, Field
from typing import Literal
import logging

logger = logging.getLogger(__name__)
app = FastAPI()

surveys_db: dict[int, dict] = {}
responses_db: dict[int, list] = {}
_survey_counter: int = 0

class SurveyCreate(BaseModel):
    title: str = Field(min_length=3, max_length=120)
    status: Literal["draft", "active"] = "draft"
    max_responses: int = Field(ge=1, le=1000, default=100)

class SurveyResponse(BaseModel):
    id: int
    title: str
    status: Literal["draft", "active", "closed"]
    max_responses: int
    response_count: int

class SubmitResponse(BaseModel):
    rating: int = Field(ge=1, le=5)
    comment: str = Field(max_length=500, default="")

class SurveyStorage:
    def create(self, data: dict) -> dict:
        global _survey_counter
        _survey_counter += 1
        record = {"id": _survey_counter, **data, "response_count": 0}
        surveys_db[_survey_counter] = record
        responses_db[_survey_counter] = []
        return record

    def get(self, survey_id: int) -> dict | None:
        return surveys_db.get(survey_id)

    def list_all(self) -> list[dict]:
        return list(surveys_db.values())

    def add_response(self, survey_id: int, response: dict) -> bool:
        survey = surveys_db.get(survey_id)
        if not survey:
            return False
        if survey["status"] != "active":
            return False
        if survey["response_count"] >= survey["max_responses"]:
            survey["status"] = "closed"
            return False
        responses_db[survey_id].append(response)
        survey["response_count"] += 1
        if survey["response_count"] >= survey["max_responses"]:
            survey["status"] = "closed"
        return True

_survey_storage = SurveyStorage()

def get_survey_storage() -> SurveyStorage:
    return _survey_storage

def notify_owner(survey_id: int, response_count: int) -> None:
    logger.info(f"Survey {survey_id} now has {response_count} responses")

@app.post("/surveys", status_code=201)
async def create_survey(
    survey: SurveyCreate,
    storage: SurveyStorage = Depends(get_survey_storage),
) -> SurveyResponse:
    created = storage.create(survey.model_dump())
    return SurveyResponse(**created)

@app.get("/surveys/{survey_id}")
async def get_survey(
    survey_id: int,
    storage: SurveyStorage = Depends(get_survey_storage),
) -> SurveyResponse:
    survey = storage.get(survey_id)
    if survey is None:
        raise HTTPException(status_code=404, detail=f"Survey {survey_id} not found")
    return SurveyResponse(**survey)

@app.get("/surveys")
async def list_surveys(
    storage: SurveyStorage = Depends(get_survey_storage),
) -> list[SurveyResponse]:
    return [SurveyResponse(**s) for s in storage.list_all()]

@app.post("/surveys/{survey_id}/respond", status_code=201)
async def submit_response(
    survey_id: int,
    response: SubmitResponse,
    background_tasks: BackgroundTasks,
    storage: SurveyStorage = Depends(get_survey_storage),
) -> dict:
    survey = storage.get(survey_id)
    if survey is None:
        raise HTTPException(status_code=404, detail=f"Survey {survey_id} not found")
    success = storage.add_response(survey_id, response.model_dump())
    if not success:
        raise HTTPException(
            status_code=409,
            detail="Survey is not accepting responses (closed or draft)"
        )
    background_tasks.add_task(notify_owner, survey_id, survey["response_count"])
    return {"status": "submitted", "survey_id": survey_id}

@app.patch("/surveys/{survey_id}/activate", status_code=200)
async def activate_survey(
    survey_id: int,
    storage: SurveyStorage = Depends(get_survey_storage),
) -> SurveyResponse:
    survey = storage.get(survey_id)
    if survey is None:
        raise HTTPException(status_code=404, detail=f"Survey {survey_id} not found")
    if survey["status"] == "closed":
        raise HTTPException(status_code=409, detail="Cannot re-activate a closed survey")
    survey["status"] = "active"
    return SurveyResponse(**survey)
```

---

**Task 3a: Implement the Core Test Suite**

Write `tests/conftest.py` and `tests/test_survey.py` that use `dependency_overrides` for isolation and cover all happy paths and all explicit error paths in every endpoint.

---

**Task 3b: Diagnose the State Isolation Gap**

After implementing 3a, examine this critically: `SurveyStorage.create()` and `SurveyStorage.add_response()` read and write `surveys_db`, `responses_db`, and `_survey_counter` — which are **module-level globals** in `survey_app.py`.

1. Does injecting a fresh `SurveyStorage()` via `dependency_overrides` actually reset `surveys_db`, `responses_db`, and `_survey_counter`? Prove your answer with a concrete execution trace.

2. If the answer is no: propose a fix that achieves true isolation **without modifying `survey_app.py`** (Rule 1). Your fix must reset all three globals before each test.

3. Without your fix in place, identify the **first specific test from 3a** that would fail when the full suite runs in order, and give the exact assertion and values.

---

**Task 3c: Add Background Task Coverage**

Add tests that verify the `notify_owner` background task is correctly scheduled (and not scheduled on failure). These tests must pass alongside the 3a suite.

**Required tests:**
1. `notify_owner` is called exactly once on a successful submission
2. `notify_owner` is called with the correct `survey_id`
3. `notify_owner` is NOT called when submission fails because the survey is in draft status

---

# Level 3 Solutions

## Solution 3a

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from survey_app import app, get_survey_storage, SurveyStorage

@pytest.fixture
def storage() -> SurveyStorage:
    return SurveyStorage()

@pytest.fixture
def client(storage: SurveyStorage) -> TestClient:
    app.dependency_overrides[get_survey_storage] = lambda: storage
    yield TestClient(app)
    app.dependency_overrides.clear()
```

```python
# tests/test_survey.py
import pytest
from fastapi.testclient import TestClient


# ── Happy paths ───────────────────────────────────────────────────────────────

def test_create_survey_returns_201_with_defaults(client):
    response = client.post("/surveys", json={"title": "Customer Feedback"})
    assert response.status_code == 201
    data = response.json()
    assert data["title"] == "Customer Feedback"
    assert data["status"] == "draft"
    assert data["response_count"] == 0
    assert data["max_responses"] == 100

def test_get_survey_returns_correct_data(client):
    create = client.post("/surveys", json={"title": "My Survey"})
    survey_id = create.json()["id"]

    response = client.get(f"/surveys/{survey_id}")
    assert response.status_code == 200
    assert response.json()["id"] == survey_id
    assert response.json()["title"] == "My Survey"

def test_list_surveys_returns_all_created(client):
    client.post("/surveys", json={"title": "Survey A"})
    client.post("/surveys", json={"title": "Survey B"})

    response = client.get("/surveys")
    assert response.status_code == 200
    assert len(response.json()) == 2

def test_activate_survey_changes_status_to_active(client):
    create = client.post("/surveys", json={"title": "Draft Survey"})
    survey_id = create.json()["id"]

    response = client.patch(f"/surveys/{survey_id}/activate")
    assert response.status_code == 200
    assert response.json()["status"] == "active"

def test_submit_to_active_survey_returns_201(client):
    create = client.post("/surveys", json={"title": "Active Survey"})
    survey_id = create.json()["id"]
    client.patch(f"/surveys/{survey_id}/activate")

    response = client.post(
        f"/surveys/{survey_id}/respond",
        json={"rating": 4, "comment": "Great!"}
    )
    assert response.status_code == 201
    assert response.json()["status"] == "submitted"
    assert response.json()["survey_id"] == survey_id

def test_survey_auto_closes_when_max_responses_reached(client):
    create = client.post("/surveys", json={"title": "Limited", "max_responses": 2})
    survey_id = create.json()["id"]
    client.patch(f"/surveys/{survey_id}/activate")

    client.post(f"/surveys/{survey_id}/respond", json={"rating": 5})
    client.post(f"/surveys/{survey_id}/respond", json={"rating": 4})

    third = client.post(f"/surveys/{survey_id}/respond", json={"rating": 3})
    assert third.status_code == 409

    get = client.get(f"/surveys/{survey_id}")
    assert get.json()["status"] == "closed"


# ── Error paths ───────────────────────────────────────────────────────────────

def test_get_nonexistent_survey_returns_404(client):
    assert client.get("/surveys/99999").status_code == 404

def test_activate_nonexistent_survey_returns_404(client):
    assert client.patch("/surveys/99999/activate").status_code == 404

def test_submit_to_nonexistent_survey_returns_404(client):
    response = client.post("/surveys/99999/respond", json={"rating": 3})
    assert response.status_code == 404

def test_submit_to_draft_survey_returns_409(client):
    create = client.post("/surveys", json={"title": "Draft Only"})
    survey_id = create.json()["id"]   # Never activated

    response = client.post(f"/surveys/{survey_id}/respond", json={"rating": 3})
    assert response.status_code == 409

def test_cannot_reactivate_closed_survey(client):
    create = client.post("/surveys", json={"title": "One Time", "max_responses": 1})
    survey_id = create.json()["id"]
    client.patch(f"/surveys/{survey_id}/activate")
    client.post(f"/surveys/{survey_id}/respond", json={"rating": 5})
    # Survey is now closed

    response = client.patch(f"/surveys/{survey_id}/activate")
    assert response.status_code == 409


# ── Validation ────────────────────────────────────────────────────────────────

@pytest.mark.parametrize("payload, expected_status", [
    pytest.param({"title": "ab"}, 422, id="title-too-short-2-chars"),
    pytest.param({"title": "x" * 121}, 422, id="title-too-long-121-chars"),
    pytest.param({"title": "Valid Title", "max_responses": 0}, 422, id="max-responses-zero"),
    pytest.param({"title": "Valid Title", "max_responses": 1001}, 422, id="max-responses-1001"),
    pytest.param({"title": "Valid Title", "status": "archived"}, 422, id="invalid-status-literal"),
    pytest.param({"title": "Minimal Valid"}, 201, id="minimal-valid-survey"),
])
def test_create_survey_validation(client, payload, expected_status):
    response = client.post("/surveys", json=payload)
    assert response.status_code == expected_status
```

---

## Solution 3b: The State Isolation Gap

**1. Does injecting a fresh `SurveyStorage()` reset the globals?**

**No.** Concrete trace:

```
1. Test 1 fixture: fresh_storage = SurveyStorage()
   → fresh_storage._notes = {}    (instance attr, unused by methods)
   → survey_app.surveys_db = {}   (module-level, unchanged by __init__)
   → survey_app._survey_counter = 0  (module-level, unchanged)

2. Test 1 calls: client.post("/surveys", json={...})
   → SurveyStorage.create() runs:
       global _survey_counter        ← refers to survey_app._survey_counter
       _survey_counter += 1          ← survey_app._survey_counter is now 1
       surveys_db[1] = record        ← survey_app.surveys_db is now {1: {...}}

3. Test 1 ends. Fixture cleanup: app.dependency_overrides.clear()
   → survey_app.surveys_db is still {1: {...}}
   → survey_app._survey_counter is still 1

4. Test 2 fixture: fresh_storage2 = SurveyStorage()
   → survey_app.surveys_db is STILL {1: {...}}   ← NOT reset!
   → survey_app._survey_counter is STILL 1       ← NOT reset!

5. Test 2 calls: client.post("/surveys", json={...})
   → _survey_counter increments to 2
   → new survey gets id=2, not id=1
```

`SurveyStorage` has no instance-level data store — its methods operate on module-level globals. A fresh instance means nothing for state isolation.

**2. Fix without modifying `survey_app.py`:**

```python
# tests/conftest.py — updated with global reset
import pytest
import survey_app
from fastapi.testclient import TestClient
from survey_app import app, get_survey_storage, SurveyStorage

@pytest.fixture(autouse=True)
def reset_survey_globals(monkeypatch):
    """
    Replace module-level globals with fresh objects before each test.
    monkeypatch restores originals automatically after the test.
    """
    monkeypatch.setattr("survey_app.surveys_db", {})
    monkeypatch.setattr("survey_app.responses_db", {})
    monkeypatch.setattr("survey_app._survey_counter", 0)

@pytest.fixture
def storage() -> SurveyStorage:
    return SurveyStorage()

@pytest.fixture
def client(storage: SurveyStorage) -> TestClient:
    app.dependency_overrides[get_survey_storage] = lambda: storage
    yield TestClient(app)
    app.dependency_overrides.clear()
```

`monkeypatch.setattr("survey_app.surveys_db", {})` replaces the module's `surveys_db` reference with a fresh empty dict for the duration of this test. When `SurveyStorage.create()` looks up `surveys_db` from the module's global namespace, it finds the replacement — clean for every test. After the test, `monkeypatch` restores the original reference automatically.

**3. First test to fail without the fix:**

`test_list_surveys_returns_all_created` fails.

This test creates 2 surveys and asserts `len(response.json()) == 2`. When tests run in order, `test_create_survey_returns_201_with_defaults` and `test_get_survey_returns_correct_data` each create surveys that accumulate in the module-level `surveys_db`. By the time `test_list_surveys_returns_all_created` runs, `surveys_db` already has entries from previous tests. After this test creates 2 more, the total is 4 (or more). The assertion fails: `AssertionError: assert 4 == 2`.

---

## Solution 3c: Background Task Coverage

```python
# tests/test_survey.py — append to existing file
from unittest.mock import patch

def test_notify_owner_called_exactly_once_on_submission(client):
    """Successful submission must schedule notify_owner exactly once."""
    create = client.post("/surveys", json={"title": "Notification Survey"})
    survey_id = create.json()["id"]
    client.patch(f"/surveys/{survey_id}/activate")

    with patch("survey_app.notify_owner") as mock_notify:
        response = client.post(
            f"/surveys/{survey_id}/respond",
            json={"rating": 3}
        )
        assert response.status_code == 201
        mock_notify.assert_called_once()


def test_notify_owner_receives_correct_survey_id(client):
    """notify_owner must be called with the survey's actual id."""
    create = client.post("/surveys", json={"title": "ID Verification Survey"})
    survey_id = create.json()["id"]
    client.patch(f"/surveys/{survey_id}/activate")

    with patch("survey_app.notify_owner") as mock_notify:
        client.post(f"/surveys/{survey_id}/respond", json={"rating": 5})

        call_args = mock_notify.call_args
        actual_survey_id = call_args.args[0]   # first positional arg
        assert actual_survey_id == survey_id


def test_notify_owner_not_called_when_submission_fails_on_draft(client):
    """If survey is in draft status, submission returns 409 and no task is scheduled."""
    create = client.post("/surveys", json={"title": "Draft Survey"})
    survey_id = create.json()["id"]
    # NOT activated — stays draft

    with patch("survey_app.notify_owner") as mock_notify:
        response = client.post(
            f"/surveys/{survey_id}/respond",
            json={"rating": 3}
        )
        assert response.status_code == 409
        mock_notify.assert_not_called()
```

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

**The Question:**

The lecture argues that testing through `TestClient` is the correct approach for FastAPI endpoints. This is true for the HTTP contract — but there are concrete scenarios where it is the wrong tool.

**Part A:** Describe a realistic scenario where `TestClient` is **not** the right tool and a different approach is more appropriate. Your scenario must be a genuine technical limitation, not personal preference, and must not be solvable by "adding a better fixture."

**Part B:** Write a minimal code example showing the scenario and why `TestClient` is wrong here:

```python
# What you are trying to test:

# Why TestClient is wrong for this purpose:

# What you would do instead:
```

**Part C:** Steelman the opposing view. For the scenario you described, explain what you lose by not using `TestClient`. What parts of the HTTP contract remain untested?

---

# Level 4 Solution

## Solution 4: Adversarial Counter-Example

**Part A: Testing complex business logic inside a dependency**

Scenario: Your app has a `PricingEngine` dependency that applies multi-tier discount logic, queries three tables, and calls an external tax service. It is injected into five different endpoints. You need to verify the discount logic is correct across twelve different input combinations.

If you test this through `TestClient`, you must: pick one endpoint to route through, construct valid HTTP payloads for that endpoint's schema, deal with routing and serialization concerns unrelated to discounts, and accept that if the endpoint's URL or input schema changes, your pricing logic tests break — even though the pricing logic itself did not change. Each of twelve combinations requires a full HTTP round-trip.

**Part B:**

```python
# What you are trying to test:
class PricingEngine:
    async def calculate(
        self, product_id: int, quantity: int, customer_tier: str
    ) -> PriceResult:
        product = await self.db.get_product(product_id)
        base = product.price * quantity
        discount = self._tier_discount(customer_tier, base)
        tax = await self.tax_service.get_rate(product.category) * (base - discount)
        return PriceResult(subtotal=base, discount=discount, tax=tax)

    def _tier_discount(self, tier: str, amount: float) -> float:
        return {"gold": 0.20, "silver": 0.10, "bronze": 0.05}.get(tier, 0.0) * amount


# Why TestClient is wrong for this purpose:
# Testing PricingEngine.calculate() through TestClient requires choosing a
# specific endpoint that uses it, constructing that endpoint's HTTP schema,
# and accepting that a URL refactor breaks these tests even though the pricing
# logic is unchanged. The twelve discount combinations become twelve HTTP
# round-trips through layers you do not care about.


# What you do instead — call the dependency function directly:
async def test_gold_tier_receives_20_percent_discount():
    fake_db = FakeProductDB()
    fake_db.add(Product(id=1, price=100.0, category="electronics"))
    fake_tax = FakeTaxService(rate=0.08)

    engine = PricingEngine(db=fake_db, tax_service=fake_tax)
    result = await engine.calculate(product_id=1, quantity=2, customer_tier="gold")

    assert result.subtotal == 200.0
    assert result.discount == pytest.approx(40.0)  # 20% of 200
    assert result.tax == pytest.approx(12.8)       # 8% of (200 - 40)
```

**Part C: What you lose by not using TestClient**

By testing `PricingEngine.calculate()` directly, the following remain untested:
- That the endpoint correctly extracts `product_id`, `quantity`, and `customer_tier` from the HTTP request
- That `PriceResult` is correctly serialized to JSON with the expected field names and types
- That the dependency injection chain from the route to `PricingEngine` is wired correctly
- That a 404 from the database propagates as an HTTP 404 (not 500)
- That auth middleware runs before pricing logic

**The correct answer is both:** test `PricingEngine`'s business logic directly to get precise, fast, isolated coverage of the complex discount rules, AND write at least one integration test through `TestClient` to verify the HTTP contract on one representative endpoint. The direct call and the `TestClient` test are complementary, not competing.

---

# LEVEL 5: THE TEACH BACK

## Scenario A: The Coverage Trophy

A junior developer submits a PR with this description:

> "I added tests for all our endpoints. We have 94% coverage, well above the 80% minimum. All 18 tests pass. The API is now well-tested."

A representative sample from their test file:

```python
def test_create_survey(client):
    response = client.post("/surveys", json={
        "title": "Customer Satisfaction Survey",
        "status": "active",
        "max_responses": 50
    })
    assert response.status_code == 201

def test_submit_response(client):
    survey = client.post("/surveys", json={"title": "Test Survey"}).json()
    client.patch(f"/surveys/{survey['id']}/activate")
    response = client.post(
        f"/surveys/{survey['id']}/respond",
        json={"rating": 4}
    )
    assert response.status_code == 201
    # Entire test body ends here — no body assertions.

def test_endpoints_do_not_crash(client):
    client.post("/surveys", json={"title": "Any Survey"})
    assert client.get("/surveys").status_code != 500
    assert client.get("/surveys/1").status_code != 500
    assert client.patch("/surveys/1/activate").status_code != 500
```

Write a code review comment that: (1) acknowledges what IS being tested, (2) explains why 94% does not mean "well-tested," (3) gives exactly 3 missing test cases that would catch real bugs, and (4) frames it as teaching, not criticism.

---

## Scenario B: The `dict.clear()` Advocate

After your review, the developer responds:

> "OK, I'll add more assertions. But why do we need `dependency_overrides`? If I call `surveys_db.clear()` and reset `_survey_counter = 0` at the start of each test, that should isolate state just as well. The override pattern feels like unnecessary complexity."

They show you this:

```python
import survey_app

@pytest.fixture(autouse=True)
def reset_state():
    survey_app.surveys_db.clear()
    survey_app.responses_db.clear()
    survey_app._survey_counter = 0
    yield

@pytest.fixture
def client():
    return TestClient(app)   # No overrides
```

Write a Slack message (3–4 paragraphs) that: (1) acknowledges the intuition is reasonable, (2) explains the specific failure mode of the `dict.clear()` approach, (3) explains what `dependency_overrides` gives you beyond state reset, (4) ends with a concrete scenario where `dict.clear()` fails and `dependency_overrides` succeeds.

---

# Level 5 Solutions

## Solution 5A: Code Review Comment

> Great start — the three tests you have here genuinely do provide value. `test_create_survey` verifies the endpoint accepts valid data and returns 201, which is the most important baseline check for any POST route. `test_submit_response` verifies the full creation-activation-submission flow works end-to-end. And `test_endpoints_do_not_crash` is a quick smoke test that catches total breakage. This is a real foundation.
>
> On the 94% number: coverage measures which lines were *executed*, not whether your assertions are meaningful. `test_endpoints_do_not_crash` executes every line in `list_surveys`, `get_survey`, and `activate_survey` — but all it checks is `!= 500`. If `activate_survey` returned `{"status": "broken"}` with a 200, the test would still pass. A test without a meaningful assertion is a test that cannot catch bugs, regardless of what it contributes to the coverage number.
>
> Here are three specific tests that would catch real bugs in the current implementation. First: submit a response to a survey that is still in **draft status** and verify you get 409, not 201. The draft-check logic is easy to accidentally delete and impossible to catch with the current suite. Second: create a survey with `max_responses=2`, submit exactly 2 responses, and assert that the survey's `status` field becomes `"closed"` and a third submission returns 409. The auto-close path is the most complex branch in the codebase and is currently completely untested. Third: in `test_submit_response`, add `assert response.json()["status"] == "submitted"` and `assert "survey_id" in response.json()`. If a developer changes the response body shape (renaming `"status"` to `"message"`), every client breaks silently — but only a body assertion would catch this before it reaches production.
>
> Happy to pair on these — it's mostly a matter of extending tests you've already written, not starting over.

---

## Solution 5B: Slack Message

> Hey! That intuition is actually very reasonable — `dict.clear()` resets the observable state and it's a natural first approach. The pattern you've shown would work for the dict contents. The immediate problem is `_survey_counter`: Python integers are immutable, so `survey_app._survey_counter = 0` in the fixture sets it at fixture setup time. But when `SurveyStorage.create()` runs `global _survey_counter; _survey_counter += 1`, it modifies the module-level variable directly. Your reset runs *before* the test, not between every individual operation, so if test A creates one survey (`_counter = 1`), your fixture resets to 0 before test B, but test B might still create a survey that gets id=1 — which is correct! Wait, actually that part works. The deeper problem is test *ordering*: if the autouse fixture runs and test B is the FIRST test that creates a survey, `_counter` is correctly 0 → new survey gets id=1. But if pytest runs tests in a different order than you expect, any test that asserts on the exact id value becomes order-sensitive.

> The bigger gap is what happens when the app gets more complex. Right now, `SurveyStorage` is a thin wrapper over module-level dicts. But imagine next sprint someone adds a `_recently_viewed_cache: dict` to track hot surveys, or adds an `audit_trail: list` that every write appends to. With `dict.clear()`, you have to find and reset every new piece of state manually — and if you miss one, tests silently share state without telling you. With `dependency_overrides`, you inject a brand-new `SurveyStorage()` instance, and everything that instance owns is fresh by definition. You don't need to know the storage's internals to get isolation.

> Here's the concrete scenario where `dict.clear()` fails: suppose we add a simple response cache inside `SurveyStorage.get()` — it caches the last-returned survey to avoid repeated dict lookups. `dict.clear()` wipes `surveys_db`, but the cache sitting on the `_survey_storage` singleton is untouched. Test A creates survey id=1, which gets cached. Your fixture clears the dict. Test B calls `GET /surveys/1` — `surveys_db` is empty, but the cache still has survey id=1, so the endpoint returns 200 with old data instead of 404. The test passes incorrectly. With `dependency_overrides` injecting `SurveyStorage()`, test B gets a fresh instance with an empty cache from the start — no stale data possible. The override is not about resetting state you know about; it's about replacing the whole object so you don't have to know.