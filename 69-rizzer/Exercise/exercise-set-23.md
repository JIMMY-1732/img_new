# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "HTTP clients are just network function calls. `requests` works anywhere Python runs. Making calls inside `async def` automatically makes them non-blocking. `raise_for_status()` plus `except httpx.HTTPStatusError` protects me from every HTTP failure. A 200 status means the response data is safe to use directly. Creating a new client per request is the clean, stateless way to manage resources. More retries means more resilient code."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | `requests.get()` inside `async def` is fine — it's just "synchronous I/O" | `requests.get()` calls a blocking socket syscall, freezing the **entire event loop thread** — every other request to every other endpoint stalls until it returns |
| 2 | `async with httpx.AsyncClient()` inside each endpoint is the "proper context manager pattern" | Each `async with` block creates a **new, empty connection pool**, pays TCP+TLS handshake overhead on every single request, then destroys the pool on exit |
| 3 | `raise_for_status()` plus `except httpx.HTTPStatusError` protects against all HTTP failures | Network-level failures (`ConnectError`, `TimeoutException`) are raised **before a response exists** — `raise_for_status()` never runs, and `httpx.HTTPStatusError` never fires |
| 4 | Sequential `await client.get(url1)` then `await client.get(url2)` is "asynchronous" — meaning concurrent | `await` means "run this coroutine and **wait here until it finishes**." Sequential awaits are sequential. Only `asyncio.gather()` achieves genuine concurrency. |
| 5 | A 200 HTTP response means the body is valid and safe to access directly | The server can return 200 with `null` fields, renamed keys, missing keys, or a completely different schema than documented. Pydantic validation is not optional. |
| 6 | Retrying all exceptions maximises resilience | Retrying permanent failures (401, 404) wastes quota. Retrying without backoff during a 503 flood sends **more traffic** to an overloaded server, preventing its recovery. |
| 7 | `timeout=10` fully protects you from hanging requests | There are four distinct timeout phases. With `timeout=10` and 3 retries, a user could wait **33+ seconds** before seeing an error. You must reason about the total time budget. |

**The Prediction Gap:** When a student sees `async def get_weather(city):` containing `requests.get(...)`, they assume the `async def` declaration makes the function non-blocking. The actual behaviour is that the entire event loop freezes for every millisecond `requests.get()` is waiting — making the `async def` declaration a lie. Students who expect non-blocking behaviour will be confidently and completely wrong.

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Async Keyword That Lied

```python
# app.py
from fastapi import FastAPI
import requests
import time

app = FastAPI()

@app.get("/weather/{city}")
async def get_weather(city: str):
    """External API takes ~3 seconds to respond."""
    response = requests.get(
        f"https://api.slowweather.com/current?city={city}",
        timeout=30,
    )
    return response.json()

@app.get("/ping")
async def ping():
    """Should respond in under 5 milliseconds — no I/O."""
    return {"status": "ok", "timestamp": time.time()}
```

A load test fires both requests **simultaneously** at `t = 0`:
- **Request A:** `GET /weather/London` — external API takes exactly 3 seconds
- **Request B:** `GET /ping` — no I/O, should be instant

**Questions:**

1. **Predict** the response time for **both** requests. When does Request B return — in milliseconds, or in ~3 seconds? Be precise.

2. **Explain why.** The developer says: *"Both functions are `async def`, which means they're non-blocking. `/ping` has no I/O, so it should return in milliseconds regardless of what `/weather` is doing."* Identify every flaw in this reasoning, step by step.

3. **Mental Simulation:** Trace the event loop's state from `t=0` to `t=3s`. After `requests.get()` is called inside `get_weather()`, what exactly is the event loop doing? Can it run `ping()` while `requests.get()` is in-flight?

4. **Fix it.** Rewrite `get_weather` so that Request B returns in milliseconds while Request A is still waiting. You must keep the external API call.

---

## Exercise 1.2: The Exception That Skips the Catcher

```python
import httpx
import asyncio

async def fetch_payment_status(transaction_id: str) -> dict:
    async with httpx.AsyncClient(timeout=5.0) as client:
        try:
            response = await client.get(
                f"https://api.payments.com/v1/transactions/{transaction_id}"
            )
            response.raise_for_status()
            return {"status": "ok", "data": response.json()}

        except httpx.HTTPStatusError as e:
            return {
                "status": "error",
                "code": e.response.status_code,
                "detail": "Payment processor returned an error",
            }
```

Consider three independent failure scenarios:

| Scenario | What Happens in the Network |
|----------|----------------------------|
| A | Server responds with `HTTP 503 Service Unavailable` |
| B | Server is completely offline — TCP connection is refused instantly |
| C | Server accepts the TCP connection but sends no response within 5 seconds |

**Questions:**

1. **Predict** for each scenario: Does the `except httpx.HTTPStatusError` block execute? What is returned, or what propagates uncaught?

2. **Explain** why Scenarios B and C cannot be caught by the current `except` block. Specifically: on which exact line does execution fail in each scenario, and what is the type of the raised exception?

3. **Mental Simulation:** In Scenario C, where does execution pause inside the `try` block? Is `raise_for_status()` ever called?

4. **Rewrite** the exception handling to correctly handle all three scenarios, returning a structured error dict with a distinct `detail` message for each failure type.

---

## Exercise 1.3: Three Seconds or One?

```python
import httpx
import asyncio
import time

async def fetch_dashboard() -> dict:
    """
    Fetches data from 3 microservices.
    Each service takes approximately 1 second to respond.
    """
    async with httpx.AsyncClient(timeout=10.0) as client:
        start = time.perf_counter()

        users_response    = await client.get("https://api.users.com/stats")
        orders_response   = await client.get("https://api.orders.com/stats")
        inventory_response = await client.get("https://api.inventory.com/stats")

        elapsed = time.perf_counter() - start
        print(f"Dashboard fetched in {elapsed:.2f}s")

        return {
            "users":     users_response.json(),
            "orders":    orders_response.json(),
            "inventory": inventory_response.json(),
        }

asyncio.run(fetch_dashboard())
```

**Questions:**

1. **Predict the output.** If each service takes exactly 1 second, does `elapsed` print approximately 1 second, 2 seconds, or 3 seconds?

2. **Explain why.** A developer argues: *"We use `async` and `await` throughout, so the three requests run asynchronously — meaning simultaneously. Each takes 1 second, so the total should be ~1 second."* Break down every word of this argument that is technically wrong.

3. **Mental Simulation:** Draw a timeline from `t=0` to completion showing:
   - Exactly when each `await client.get(...)` starts
   - Exactly when each `await client.get(...)` returns
   - Whether the event loop is free to handle other coroutines during the 1-second waits

4. **Rewrite** `fetch_dashboard()` so all three requests run genuinely concurrently. The total time should now be approximately 1 second, not 3.

---

## Exercise 1.4: The 200 That Lied

```python
import httpx
import asyncio

async def build_user_card(username: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://api.socialnetwork.com/users/{username}"
        )
        response.raise_for_status()

        # The API returned 200 — this is safe now, right?
        data = response.json()

        return {
            "display_name": data["name"].upper(),
            "handle":       f"@{data['username']}",
            "bio_preview":  data["bio"][:100],
            "website":      data["links"]["homepage"],
            "follower_count": f"{data['followers']:,}",
        }
```

All three of these users **exist** — the API returns `HTTP 200` for all of them:

```json
// active_user — all fields populated
{"name": "Jane Smith", "username": "active_user", "bio": "Engineer", "links": {"homepage": "https://jane.dev"}, "followers": 1500}

// new_user — some fields null
{"name": null, "username": "new_user", "bio": null, "links": {"homepage": null}, "followers": 0}

// minimal_user — "links" key missing entirely
{"name": "Min", "username": "minimal_user", "bio": "Hello", "followers": 42}
```

**Questions:**

1. **Predict** the outcome for all three `build_user_card()` calls. For each: either write the exact return value, or identify the exact exception type and the exact line where it is raised.

2. **Explain the gap.** Why does `raise_for_status()` not protect against these failures? What is the difference between an HTTP status code and data validity?

3. **Systematic audit.** For each line of the `return` block, classify it:
   - ✅ **Safe** — cannot fail regardless of the data
   - ❌ **Unsafe — None** — fails if the value is `None`
   - ❌ **Unsafe — Missing key** — fails if the key is absent
   - ❌ **Unsafe — Wrong type** — fails if the value is an unexpected type

4. **Rewrite** using a Pydantic model that handles all three user types without crashing. On validation failure, raise a custom `ExternalAPIError`.

---

# Level 1 Solutions

## Solution 1.1: The Async Keyword That Lied

**1. Predicted response times:**
- Request A (`/weather/London`): returns at approximately `t = 3.0s`
- Request B (`/ping`): returns at approximately `t = 3.0s` — **not milliseconds**

**2. What is wrong with the reasoning:**

The reasoning has one fatal flaw and one superficially true but irrelevant point.

The superficially true part: both are `async def`, which makes them coroutines. But `async def` only declares the function's *type* — it says nothing about what happens *inside* it.

The fatal flaw: `requests.get()` is a synchronous function. Internally it calls `socket.recv()`, a blocking OS syscall. When the OS thread blocks on `socket.recv()`, the asyncio event loop — which runs on that same OS thread — is also blocked. It cannot switch to any other coroutine. `async def` is a lie when you put a blocking call inside it.

The distinction is:
- `await asyncio.sleep(3)` → coroutine **suspends**, event loop runs other tasks ✅
- `requests.get(url)` → OS thread **blocks**, event loop is frozen ❌

**3. Mental simulation:**

```
t=0.0s: Event loop starts get_weather() coroutine
        requests.get() called → OS thread enters blocking socket wait
        ── EVENT LOOP IS FROZEN ─────────────────────────────────
t=0.0s: Request B arrives → queued, cannot execute
t=3.0s: External API responds → socket.recv() returns, OS thread unblocks
        ── EVENT LOOP UNFROZEN ──────────────────────────────────
t=3.0s: get_weather() returns its response
t=3.0s: Event loop finally processes Request B → ping() runs instantly
t=3.0s: ping() returns
```

**4. The fix:**

```python
import httpx

@app.get("/weather/{city}")
async def get_weather(city: str):
    async with httpx.AsyncClient(timeout=30.0) as client:
        # `await` SUSPENDS the coroutine here.
        # The event loop is FREE to run ping() while this waits.
        response = await client.get(
            f"https://api.slowweather.com/current?city={city}"
        )
    return response.json()
```

With `await client.get()`, the coroutine suspends at the `await` keyword and the event loop can process `ping()` immediately. Request B now returns in milliseconds.

**Key Insight:** `async def` + `await non-blocking I/O` = non-blocking. `async def` + synchronous blocking call = event loop frozen. The `async` keyword on the function declaration is meaningless if the body contains synchronous blocking code.

---

## Solution 1.2: The Exception That Skips the Catcher

**1. Scenario predictions:**

| Scenario | `except httpx.HTTPStatusError` fires? | Outcome |
|----------|--------------------------------------|---------|
| A (503 response) | ✅ Yes | Returns error dict with `code: 503` |
| B (connection refused) | ❌ No | `httpx.ConnectError` propagates uncaught |
| C (5-second timeout) | ❌ No | `httpx.ReadTimeout` propagates uncaught |

**2. Why B and C bypass the except block:**

`httpx.HTTPStatusError` is only raised by `response.raise_for_status()`. That method requires a `response` object to call it on. In Scenarios B and C, no response object ever exists:

- **Scenario B:** The TCP handshake fails immediately. There is no HTTP exchange whatsoever. `httpx` raises `httpx.ConnectError` (a subclass of `httpx.RequestError`) directly at the `await client.get(...)` line.
- **Scenario C:** The TCP connection is established, but the server sends no data within 5 seconds. `httpx` raises `httpx.ReadTimeout` (a subclass of `httpx.TimeoutException`) at the `await client.get(...)` line.

In both cases, execution jumps out of the `try` block before `raise_for_status()` is ever reached. The `except httpx.HTTPStatusError` clause has nothing to catch.

**3. Mental simulation for Scenario C:**

```python
response = await client.get(...)    # ← Execution STOPS HERE at t=5s, raises ReadTimeout
response.raise_for_status()         # ← NEVER REACHED
return {"status": "ok", ...}        # ← NEVER REACHED
```

**4. Correct rewrite:**

```python
async def fetch_payment_status(transaction_id: str) -> dict:
    async with httpx.AsyncClient(timeout=5.0) as client:
        try:
            response = await client.get(
                f"https://api.payments.com/v1/transactions/{transaction_id}"
            )
            response.raise_for_status()
            return {"status": "ok", "data": response.json()}

        except httpx.ConnectError:
            return {"status": "error", "detail": "Payment processor is unreachable"}

        except httpx.ReadTimeout:
            return {"status": "error", "detail": "Payment processor timed out"}

        except httpx.TimeoutException:
            # Catch-all for WriteTimeout, PoolTimeout
            return {"status": "error", "detail": "Request timed out"}

        except httpx.HTTPStatusError as e:
            return {
                "status": "error",
                "code": e.response.status_code,
                "detail": f"Payment processor returned {e.response.status_code}",
            }
```

**Key Insight:** The httpx exception hierarchy has two completely separate branches. `httpx.HTTPStatusError` only covers "server responded, but with an error status." `httpx.RequestError` covers "could not complete the request at the network level." You must handle both branches. `raise_for_status()` only bridges the first.

---

## Solution 1.3: Three Seconds or One?

**1. Predicted output:** approximately **3.00 seconds**.

**2. Breaking down the flawed argument word by word:**

- *"We use `async` and `await` throughout"* — True, but irrelevant to concurrency.
- *"so the three requests run asynchronously"* — `asynchronously` here is being misused to mean `simultaneously`. It means neither. It means each call *can* yield control — but only does so one at a time.
- *"meaning simultaneously"* — **False.** `await` means "pause here and wait for this one thing to complete before moving on."
- *"Each takes 1 second, so the total should be ~1 second"* — **False.** That would be true only if all three were running at the same time, which requires `asyncio.gather()`.

**3. Timeline:**

```
t=0.0s: await client.get("...users...")    ← starts, suspends coroutine
t=1.0s: users response arrives             ← coroutine resumes
t=1.0s: await client.get("...orders...")   ← starts, suspends coroutine
t=2.0s: orders response arrives            ← coroutine resumes
t=2.0s: await client.get("...inventory".) ← starts, suspends coroutine
t=3.0s: inventory response arrives         ← coroutine resumes
t=3.0s: return result
```

The event loop is free to handle *other* incoming requests during each 1-second pause. But the three fetches are entirely sequential with respect to each other.

**4. The fix:**

```python
async def fetch_dashboard() -> dict:
    async with httpx.AsyncClient(timeout=10.0) as client:
        start = time.perf_counter()

        # All three coroutines START simultaneously.
        # asyncio.gather() suspends until ALL THREE complete.
        users_resp, orders_resp, inventory_resp = await asyncio.gather(
            client.get("https://api.users.com/stats"),
            client.get("https://api.orders.com/stats"),
            client.get("https://api.inventory.com/stats"),
        )

        elapsed = time.perf_counter() - start
        print(f"Dashboard fetched in {elapsed:.2f}s")  # ~1.0s

        return {
            "users":     users_resp.json(),
            "orders":    orders_resp.json(),
            "inventory": inventory_resp.json(),
        }
```

**Key Insight:** `await` is sequential. `asyncio.gather()` is concurrent. Every `await` you write is a checkpoint where the event loop may switch to another coroutine, but it does not mean multiple things are running simultaneously. You must explicitly request concurrency with `asyncio.gather()`.

---

## Solution 1.4: The 200 That Lied

**1. Predicted outcomes:**

| User | Outcome |
|------|---------|
| `active_user` | Returns successfully — all fields non-null |
| `new_user` | `AttributeError: 'NoneType' object has no attribute 'upper'` at `data["name"].upper()` |
| `minimal_user` | `KeyError: 'links'` at `data["links"]["homepage"]` |

**2. The gap between HTTP 200 and valid data:**

`raise_for_status()` inspects only the **status code** — the envelope. It verifies the server processed the request and chose to respond. It says nothing about whether the response body matches any schema. The server can truthfully report "200 OK, here is the user" while that user has null fields, optional keys omitted, or a structure different from documentation.

**3. Systematic audit:**

```python
return {
    "display_name": data["name"].upper(),        # ❌ Unsafe — None: AttributeError if name=null
    "handle":       f"@{data['username']}",      # ✅ Safe: always present in all three responses
    "bio_preview":  data["bio"][:100],           # ❌ Unsafe — None: TypeError if bio=null
    "website":      data["links"]["homepage"],   # ❌ Unsafe — Missing key: KeyError if "links" absent
                                                 # ❌ Also Unsafe — None: the value could be null too
    "follower_count": f"{data['followers']:,}",  # ⚠️ Probably safe, but no guarantee it's int
}
```

**4. Pydantic rewrite:**

```python
from pydantic import BaseModel, ValidationError
from typing import Optional

class SocialLinks(BaseModel):
    homepage: Optional[str] = None
    model_config = {"extra": "ignore"}

class SocialUser(BaseModel):
    name: Optional[str] = None
    username: str
    bio: Optional[str] = None
    links: Optional[SocialLinks] = None
    followers: int = 0
    model_config = {"extra": "ignore"}

class ExternalAPIError(Exception):
    pass

async def build_user_card(username: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://api.socialnetwork.com/users/{username}"
        )
        response.raise_for_status()

        try:
            user = SocialUser.model_validate(response.json())
        except ValidationError as e:
            raise ExternalAPIError(f"Unexpected API response structure: {e}") from e

        return {
            "display_name":   user.name.upper() if user.name else "Anonymous",
            "handle":         f"@{user.username}",
            "bio_preview":    user.bio[:100] if user.bio else "",
            "website":        user.links.homepage if user.links else None,
            "follower_count": f"{user.followers:,}",
        }
```

**Key Insight:** Treat every external API response as untrusted input — exactly like incoming request bodies. The pattern is identical to what you already know: `response.json()` → `Model.model_validate(raw_data)`. If GitHub changes their API and renames `name` to `display_name`, Pydantic catches it immediately with a `ValidationError` that tells you *exactly which field broke*. Without it, you get an `AttributeError` somewhere deep in your business logic hours later.

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Connection Pool That Isn't

**Context:** A developer built a three-endpoint weather service. It is functionally correct — all responses are accurate, all errors are handled. During development with one or two users, it works fine. Under load testing at 50 requests/second, it's significantly slower than expected and performance degrades further as traffic increases.

```python
# weather_api.py
from fastapi import FastAPI
import httpx
import asyncio

app = FastAPI()
API_KEY = "your-api-key"
BASE_URL = "https://api.openweathermap.org/data/2.5"

@app.get("/weather/current/{city}")
async def get_current(city: str):
    async with httpx.AsyncClient(
        timeout=httpx.Timeout(connect=5.0, read=10.0, write=5.0, pool=5.0),
        headers={"Accept": "application/json"},
    ) as client:
        response = await client.get(
            f"{BASE_URL}/weather",
            params={"q": city, "appid": API_KEY, "units": "metric"},
        )
        response.raise_for_status()
        return response.json()


@app.get("/weather/forecast/{city}")
async def get_forecast(city: str):
    async with httpx.AsyncClient(
        timeout=httpx.Timeout(connect=5.0, read=10.0, write=5.0, pool=5.0),
        headers={"Accept": "application/json"},
    ) as client:
        response = await client.get(
            f"{BASE_URL}/forecast",
            params={"q": city, "appid": API_KEY, "units": "metric"},
        )
        response.raise_for_status()
        return response.json()


@app.get("/weather/full/{city}")
async def get_full(city: str):
    async with httpx.AsyncClient(
        timeout=httpx.Timeout(connect=5.0, read=10.0, write=5.0, pool=5.0),
        headers={"Accept": "application/json"},
    ) as client:
        current, forecast = await asyncio.gather(
            client.get(f"{BASE_URL}/weather",
                       params={"q": city, "appid": API_KEY, "units": "metric"}),
            client.get(f"{BASE_URL}/forecast",
                       params={"q": city, "appid": API_KEY, "units": "metric"}),
        )
        current.raise_for_status()
        forecast.raise_for_status()
        return {"current": current.json(), "forecast": forecast.json()}
```

---

**Task 2a: Identify the Flaw**

The code is functionally correct. Identify the specific performance flaw. Point to the exact mechanism and the exact line responsible. Do not fix the code yet.

---

**Task 2b: Explain the Mechanism**

`api.openweathermap.org` uses HTTPS.

1. Draw a simplified network timeline showing what happens at the TCP and TLS layers for the **first** request vs the **tenth** request to `/weather/current/London` with the current code.
2. What specific protocol operations are paid on **every single request** because of the code structure?
3. Why does the overhead worsen relative to useful work as requests per second increases?

---

**Task 2c: Instrument to Prove Diagnosis**

Add timing instrumentation that makes the handshake overhead visible. Your output should clearly show the difference between the overhead paid on a "cold" request vs what that overhead could be on a "warm" request.

```python
# Your instrumented version should produce output like:
#
# [COLD] London: total=248ms, http_transfer=81ms, overhead=167ms
# [COLD] Paris:  total=251ms, http_transfer=79ms, overhead=172ms
#
# If fixed:
# [COLD] London: total=248ms, http_transfer=81ms, overhead=167ms
# [WARM] Paris:  total=86ms,  http_transfer=82ms, overhead=4ms
```

Write the instrumented version and show the expected output pattern.

---

**Task 2d: Fix the Code**

Refactor using the FastAPI `lifespan` pattern and `Depends()` so that:
1. One `AsyncClient` is created at startup and shared across all three endpoints
2. The client is properly closed at shutdown
3. All three endpoints still function identically
4. The pattern mirrors the database session dependency you used in Week 6

---

**Task 2e: Write a Verification Test**

Write a test that:
- Passes on your fixed code
- Would **fail** if someone reverted to creating a new `AsyncClient` per request
- Uses `dependency_overrides` or a mock transport to avoid real network calls

```python
# Hint: if the client is truly shared, the same Python object
# must service all requests. How do you assert object identity
# across multiple endpoint calls?
```

---

## Exercise 2.2: The Retry That Makes Everything Worse

**Context:** After a 20-minute outage, an engineer added tenacity retry logic to all GitHub API calls. During the *next* outage, GitHub's API took significantly longer to recover than it normally would. The engineer is confused — the retries were supposed to help.

```python
# github_client.py
from tenacity import (
    retry, stop_after_attempt,
    wait_exponential_jitter,
    retry_if_exception_type,
)
import httpx

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential_jitter(initial=0.5, max=5),
    retry=retry_if_exception_type(Exception),  # Retry on ANYTHING
)
async def get_repository(
    client: httpx.AsyncClient, owner: str, repo: str
) -> dict:
    response = await client.get(f"https://api.github.com/repos/{owner}/{repo}")
    response.raise_for_status()
    return response.json()


@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential_jitter(initial=0.5, max=5),
    retry=retry_if_exception_type(Exception),  # Retry on ANYTHING
)
async def get_user(
    client: httpx.AsyncClient, username: str
) -> dict:
    response = await client.get(f"https://api.github.com/users/{username}")
    response.raise_for_status()
    return response.json()
```

---

**Task 2a: Identify the Flaw(s)**

There are at least **three distinct problems** with this retry configuration. Identify all three with a single precise sentence each. Consider: what happens when the repo doesn't exist? When your API key is invalid? When `reraise` is not configured?

---

**Task 2b: Explain the Mechanism**

For each problem you found in 2a, describe the specific harm it causes. For the "retrying 401" case specifically, calculate the exact wall-clock time wasted and the total number of unnecessary network requests made.

---

**Task 2c: Show the Thundering Herd**

GitHub begins a 60-second maintenance window, returning `HTTP 503` for all API calls. Your application has 50 active users simultaneously triggering `get_repository()`.

1. Describe the sequence of request batches hitting GitHub's servers during the maintenance window.
2. Calculate the **total** HTTP requests sent to GitHub during those 60 seconds, given 50 users × 5 attempts maximum.
3. Explain why this sequence makes recovery **harder** for GitHub, not easier.

---

**Task 2d: Fix the Code**

Rewrite both functions with correct retry configuration:
- Retry on: `httpx.TimeoutException`, `httpx.ConnectError`, HTTP 429, HTTP 5xx
- Do **not** retry on: any other 4xx status code
- Preserve the original exception on final failure (not `tenacity.RetryError`)

---

**Task 2e: Write Verification Tests**

```python
import pytest
import httpx
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_404_does_not_retry():
    """A 404 must raise immediately — exactly 1 HTTP call made."""
    call_count = 0

    async def mock_get(*args, **kwargs):
        nonlocal call_count
        call_count += 1
        return httpx.Response(404, json={"message": "Not Found"})

    mock_client = AsyncMock(spec=httpx.AsyncClient)
    mock_client.get = mock_get

    with pytest.raises(httpx.HTTPStatusError):
        await get_repository(mock_client, "ghost", "nonexistent")

    assert call_count == 1, f"404 should not retry. Got {call_count} calls."


@pytest.mark.asyncio
async def test_503_retries_to_exhaustion():
    """Persistent 503 should retry up to max_attempts then raise."""
    call_count = 0

    async def mock_get(*args, **kwargs):
        nonlocal call_count
        call_count += 1
        return httpx.Response(503, json={"message": "Service Unavailable"})

    mock_client = AsyncMock(spec=httpx.AsyncClient)
    mock_client.get = mock_get

    with pytest.raises(httpx.HTTPStatusError) as exc_info:
        await get_repository(mock_client, "owner", "repo")

    assert call_count == 5
    assert exc_info.value.response.status_code == 503  # Not RetryError!


@pytest.mark.asyncio
async def test_timeout_then_success_retries_correctly():
    """Two timeouts followed by success → succeeds, 3 total calls made."""
    call_count = 0

    async def mock_get(*args, **kwargs):
        nonlocal call_count
        call_count += 1
        if call_count < 3:
            raise httpx.ReadTimeout("Timed out", request=None)
        return httpx.Response(200, json={"name": "repo", "full_name": "owner/repo"})

    mock_client = AsyncMock(spec=httpx.AsyncClient)
    mock_client.get = mock_get

    result = await get_repository(mock_client, "owner", "repo")
    assert result["name"] == "repo"
    assert call_count == 3
```

---

# Level 2 Solutions

## Solution 2.1: The Connection Pool That Isn't

**Task 2a: The Flaw**

Every HTTP request pays the full TCP+TLS handshake cost from scratch. The flaw is structural: `async with httpx.AsyncClient() as client:` creates a new `AsyncClient` instance with an empty connection pool. When the `async with` block exits at the bottom of each endpoint function, `aclose()` is called, shutting down every open connection. The next request creates a new client and pays for a new handshake. No connections are ever reused.

---

**Task 2b: The Mechanism**

Request 1 and Request 10 are identical in cost under the current code:

```
Every request (current code):
  DNS resolution:      ~20ms  (may be OS-cached after first)
  TCP 3-way handshake: ~50ms  (SYN → SYN-ACK → ACK)
  TLS 1.3 handshake:  ~100ms  (ClientHello, ServerHello, Certificate, Finished)
  HTTP GET + response: ~80ms
  ──────────────────────────
  Total:              ~250ms  Connection then DESTROYED by aclose()

Same cost for request 2, 3, 4... every single one.
```

With a shared client, connection 1 is established once and kept open (HTTP keep-alive):

```
Request 1: ~250ms (handshakes + HTTP)
Request 2:  ~80ms (connection reused — no handshake!)
Request 3:  ~80ms
...
```

Under 50 req/s to this endpoint, the broken version pays `50 × 170ms = 8,500ms` of pure handshake overhead per second. The fixed version pays that once. As traffic scales, the difference becomes the dominant performance cost.

---

**Task 2c: Instrumentation**

```python
import time

@app.get("/weather/current/{city}")
async def get_current(city: str):
    wall_start = time.perf_counter()

    async with httpx.AsyncClient(
        timeout=httpx.Timeout(connect=5.0, read=10.0, write=5.0, pool=5.0),
    ) as client:
        response = await client.get(
            f"{BASE_URL}/weather",
            params={"q": city, "appid": API_KEY, "units": "metric"},
        )
        response.raise_for_status()

    wall_ms = (time.perf_counter() - wall_start) * 1000
    http_ms = response.elapsed.total_seconds() * 1000
    overhead_ms = wall_ms - http_ms

    print(
        f"[{'COLD':4}] {city}: "
        f"total={wall_ms:.0f}ms, "
        f"http_transfer={http_ms:.0f}ms, "
        f"overhead={overhead_ms:.0f}ms"
    )
    return response.json()

# Expected output pattern (broken version — every request shows high overhead):
# [COLD] London: total=248ms, http_transfer=81ms, overhead=167ms
# [COLD] Paris:  total=252ms, http_transfer=79ms, overhead=173ms
# [COLD] Tokyo:  total=245ms, http_transfer=80ms, overhead=165ms
#
# Expected output pattern (fixed version — overhead only on first request):
# [COLD] London: total=248ms, http_transfer=81ms, overhead=167ms
# [WARM] Paris:  total=86ms,  http_transfer=83ms, overhead=3ms
# [WARM] Tokyo:  total=84ms,  http_transfer=81ms, overhead=3ms
```

---

**Task 2d: The Fix**

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends, Request
import httpx, asyncio

WEATHER_TIMEOUT = httpx.Timeout(connect=5.0, read=10.0, write=5.0, pool=5.0)
WEATHER_LIMITS  = httpx.Limits(max_connections=20, max_keepalive_connections=10)

@asynccontextmanager
async def lifespan(app: FastAPI):
    # ONE client for the lifetime of the application
    app.state.weather_client = httpx.AsyncClient(
        base_url="https://api.openweathermap.org/data/2.5",
        timeout=WEATHER_TIMEOUT,
        limits=WEATHER_LIMITS,
        headers={"Accept": "application/json"},
    )
    yield
    await app.state.weather_client.aclose()

app = FastAPI(lifespan=lifespan)

async def get_weather_client(request: Request) -> httpx.AsyncClient:
    return request.app.state.weather_client

@app.get("/weather/current/{city}")
async def get_current(
    city: str,
    client: httpx.AsyncClient = Depends(get_weather_client),
):
    response = await client.get(
        "/weather", params={"q": city, "appid": API_KEY, "units": "metric"}
    )
    response.raise_for_status()
    return response.json()

@app.get("/weather/forecast/{city}")
async def get_forecast(
    city: str,
    client: httpx.AsyncClient = Depends(get_weather_client),
):
    response = await client.get(
        "/forecast", params={"q": city, "appid": API_KEY, "units": "metric"}
    )
    response.raise_for_status()
    return response.json()

@app.get("/weather/full/{city}")
async def get_full(
    city: str,
    client: httpx.AsyncClient = Depends(get_weather_client),
):
    current, forecast = await asyncio.gather(
        client.get("/weather",   params={"q": city, "appid": API_KEY, "units": "metric"}),
        client.get("/forecast",  params={"q": city, "appid": API_KEY, "units": "metric"}),
    )
    current.raise_for_status()
    forecast.raise_for_status()
    return {"current": current.json(), "forecast": forecast.json()}
```

---

**Task 2e: Verification Test**

```python
import pytest
from httpx import AsyncClient
from fastapi import Request

@pytest.mark.asyncio
async def test_client_is_shared_across_requests():
    """
    The SAME AsyncClient object must handle all requests.
    If a new client is created per-request, this test fails.
    """
    observed_client_ids: list[int] = []

    async def spy_on_client(request: Request) -> httpx.AsyncClient:
        client = request.app.state.weather_client
        observed_client_ids.append(id(client))
        return client

    app.dependency_overrides[get_weather_client] = spy_on_client

    async with AsyncClient(app=app, base_url="http://test") as ac:
        # Three separate requests — each should receive the same client object
        for city in ["London", "Paris", "Tokyo"]:
            # We don't care if these fail (no real API key in tests)
            # We only care about which client object was injected
            try:
                await ac.get(f"/weather/current/{city}")
            except Exception:
                pass

    app.dependency_overrides.clear()

    unique_clients = set(observed_client_ids)
    assert len(unique_clients) == 1, (
        f"Expected 1 unique client instance across all requests, "
        f"got {len(unique_clients)}. Client is being recreated per request."
    )
```

---

## Solution 2.2: The Retry That Makes Everything Worse

**Task 2a: Three Distinct Problems**

1. `retry_if_exception_type(Exception)` retries **all exceptions including permanent failures** — 404, 401, and 403 will all retry, despite being conditions that cannot change between attempts.

2. `raise_for_status()` is called inside the retried function, which means `httpx.HTTPStatusError` (for any 4xx/5xx) is caught by `retry_if_exception_type(Exception)`. The retry decorator sees no distinction between a retryable 503 and a permanent 401.

3. `reraise=True` is not set, so after all attempts are exhausted, tenacity raises `RetryError` instead of the original exception. Callers receive a `RetryError` with no information about whether the final failure was a 404, a 503, or a timeout.

---

**Task 2b: Mechanism**

**Retrying 401 (invalid API key) — exact timeline:**

```
t=0.0s:  Attempt 1: GET /repos/... → 401 Unauthorized
         raise_for_status() → HTTPStatusError
         retry_if_exception_type(Exception) → True → wait ~0.5s
t=0.5s:  Attempt 2: GET /repos/... → 401 (key is still invalid)
         → wait ~1s
t=1.5s:  Attempt 3: GET /repos/... → 401
         → wait ~2s
t=3.5s:  Attempt 4: GET /repos/... → 401
         → wait ~4s
t=7.5s:  Attempt 5: GET /repos/... → 401
         All attempts exhausted → raises RetryError (not HTTPStatusError!)

Wasted: 5 network requests, ~7.5 seconds, caller gets RetryError
with no indication the problem was an invalid API key.
```

**Retrying 404 (repo doesn't exist):** Same outcome. 5 requests. The repository's non-existence does not change between retries.

**Effect on rate limits:** GitHub's API limits are per-API-key per hour. Every unnecessary 401 or 404 retry consumes quota against a problem that retrying cannot fix.

---

**Task 2c: The Thundering Herd**

```
t=0s:    50 users trigger get_repository() simultaneously
         50 requests hit GitHub → all 503
t=~0.5s: 50 retry 1 → 50 more requests (GitHub still in maintenance)
t=~1.5s: 50 retry 2 → 50 more requests
t=~3.5s: 50 retry 3 → 50 more requests
t=~7.5s: 50 retry 4 → 50 more requests
         All 5 attempts exhausted

Total requests to GitHub: 50 × 5 = 250 in ~7.5 seconds
```

**Why this prevents recovery:** GitHub entered maintenance because it was overloaded at 50 requests. While it's attempting to restart, it receives 250 requests — 5× the original load. Recovery requires traffic to drop. Your retry logic ensures traffic stays high. Instead of a brief outage, you've contributed to an extended one.

The correct tool here is a **circuit breaker** (Week 8, Lecture 2): after detecting N consecutive 503s, stop calling the service entirely and return a fast error for a cooldown period. Give the server room to breathe.

---

**Task 2d: Fixed Code**

```python
from tenacity import (
    retry, stop_after_attempt, wait_exponential_jitter,
    retry_if_exception_type, retry_if_result,
    before_sleep_log, RetryError,
)
import httpx
import logging

logger = logging.getLogger(__name__)

RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 504}

def _is_retryable_response(response: httpx.Response) -> bool:
    return response.status_code in RETRYABLE_STATUS_CODES


@retry(
    stop=stop_after_attempt(4),
    wait=wait_exponential_jitter(initial=1, max=15),
    retry=(
        retry_if_exception_type((httpx.TimeoutException, httpx.ConnectError))
        | retry_if_result(_is_retryable_response)
    ),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,  # Raise original exception, not RetryError
)
async def _raw_github_fetch(
    client: httpx.AsyncClient, path: str
) -> httpx.Response:
    """
    Low-level GitHub fetch with retry.
    Returns the response — does NOT call raise_for_status.
    Tenacity inspects the response via _is_retryable_response.
    """
    return await client.get(f"https://api.github.com{path}")


async def get_repository(
    client: httpx.AsyncClient, owner: str, repo: str
) -> dict:
    response = await _raw_github_fetch(client, f"/repos/{owner}/{repo}")
    response.raise_for_status()  # Now raises for non-retried 4xx errors
    return response.json()


async def get_user(
    client: httpx.AsyncClient, username: str
) -> dict:
    response = await _raw_github_fetch(client, f"/users/{username}")
    response.raise_for_status()
    return response.json()
```

---

**Task 2e: Verification Tests — Correct Versions**

```python
import pytest
import httpx
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_404_does_not_retry():
    call_count = 0

    async def mock_get(*args, **kwargs):
        nonlocal call_count
        call_count += 1
        return httpx.Response(404, json={"message": "Not Found"})

    mock_client = AsyncMock(spec=httpx.AsyncClient)
    mock_client.get = mock_get

    with pytest.raises(httpx.HTTPStatusError) as exc_info:
        await get_repository(mock_client, "ghost", "nonexistent")

    assert call_count == 1, f"404 should NOT retry. Got {call_count} calls."
    assert exc_info.value.response.status_code == 404  # Not RetryError


@pytest.mark.asyncio
async def test_503_retries_to_exhaustion():
    call_count = 0

    async def mock_get(*args, **kwargs):
        nonlocal call_count
        call_count += 1
        return httpx.Response(503, json={"message": "Service Unavailable"})

    mock_client = AsyncMock(spec=httpx.AsyncClient)
    mock_client.get = mock_get

    with pytest.raises(httpx.HTTPStatusError) as exc_info:
        await get_repository(mock_client, "owner", "repo")

    assert call_count == 4  # 4 attempts (stop_after_attempt(4))
    assert exc_info.value.response.status_code == 503  # reraise=True preserves original


@pytest.mark.asyncio
async def test_timeout_then_success_retries_correctly():
    call_count = 0

    async def mock_get(*args, **kwargs):
        nonlocal call_count
        call_count += 1
        if call_count < 3:
            raise httpx.ReadTimeout("Timed out", request=None)
        return httpx.Response(200, json={"name": "repo", "full_name": "owner/repo"})

    mock_client = AsyncMock(spec=httpx.AsyncClient)
    mock_client.get = mock_get

    result = await get_repository(mock_client, "owner", "repo")
    assert result["name"] == "repo"
    assert call_count == 3  # 2 timeouts + 1 success
```

---

# LEVEL 2.5: EXTEND EXISTING CODE

**Background:** The following is a working, correct GitHub user service. The lifespan and dependency injection patterns are already in place. All provided tests pass. Do not break them.

```python
# github_service.py
from contextlib import asynccontextmanager
from fastapi import FastAPI, HTTPException, Depends, Request
from pydantic import BaseModel, ValidationError
from typing import Optional
import httpx

class GitHubUser(BaseModel):
    login: str
    name: Optional[str] = None
    bio: Optional[str] = None
    public_repos: int
    followers: int
    model_config = {"extra": "ignore"}

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.github_client = httpx.AsyncClient(
        base_url="https://api.github.com",
        headers={"Accept": "application/vnd.github.v3+json"},
        timeout=5.0,          # ← Simple uniform timeout
    )
    yield
    await app.state.github_client.aclose()

app = FastAPI(lifespan=lifespan)

async def get_github_client(request: Request) -> httpx.AsyncClient:
    return request.app.state.github_client

@app.get("/users/{username}", response_model=GitHubUser)
async def get_user(
    username: str,
    client: httpx.AsyncClient = Depends(get_github_client),
):
    try:
        response = await client.get(f"/users/{username}")
        response.raise_for_status()
        return GitHubUser.model_validate(response.json())
    except httpx.HTTPStatusError as e:
        if e.response.status_code == 404:
            raise HTTPException(status_code=404, detail=f"User '{username}' not found")
        raise HTTPException(status_code=502, detail="GitHub API error")
    except httpx.RequestError:
        raise HTTPException(status_code=502, detail="Could not reach GitHub")
```

**Provided Tests (must pass after ALL three extensions):**

```python
# test_github_service.py
import pytest
from httpx import AsyncClient, Response
from unittest.mock import AsyncMock

@pytest.fixture
def mock_github_app(monkeypatch):
    """Returns the app with GitHub calls mocked out."""
    async def mock_get(path, **kwargs):
        if "torvalds" in path:
            return Response(200, json={
                "login": "torvalds", "name": "Linus Torvalds",
                "bio": "Just a kernel hacker", "public_repos": 5, "followers": 200000,
            })
        if "nonexistent" in path:
            return Response(404, json={"message": "Not Found"})
        raise httpx.ConnectError("Unreachable")

    mock_client = AsyncMock(spec=httpx.AsyncClient)
    mock_client.get = mock_get
    app.dependency_overrides[get_github_client] = lambda: mock_client
    yield app
    app.dependency_overrides.clear()


@pytest.mark.asyncio
async def test_successful_fetch_returns_user(mock_github_app):
    async with AsyncClient(app=mock_github_app, base_url="http://test") as ac:
        response = await ac.get("/users/torvalds")
    assert response.status_code == 200
    assert response.json()["login"] == "torvalds"

@pytest.mark.asyncio
async def test_404_returns_http_404(mock_github_app):
    async with AsyncClient(app=mock_github_app, base_url="http://test") as ac:
        response = await ac.get("/users/nonexistent")
    assert response.status_code == 404

@pytest.mark.asyncio
async def test_network_error_returns_502(mock_github_app):
    async with AsyncClient(app=mock_github_app, base_url="http://test") as ac:
        response = await ac.get("/users/unreachable")
    assert response.status_code == 502
```

---

**Extension 1: Replace Uniform Timeout with Differentiated Timeout Handling**

The current `timeout=5.0` is too aggressive on the