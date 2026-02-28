## LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM FIRST, TOOL LAST                                       │
│  ────────────────────────                                       │
│  Students must FEEL why testing a function directly fails       │
│  before we hand them TestClient. Let them try and break it.     │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  API testing is abstract. We use a vending machine analogy      │
│  throughout. Every concept maps to something you can touch.     │
│                                                                 │
│  BUILD ON WHAT THEY KNOW                                        │
│  ───────────────────────                                        │
│  Week 2's pytest is the foundation. We are NOT teaching         │
│  testing from scratch. We are teaching a NEW CONTEXT            │
│  for skills they already have.                                  │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  pytest (Week 2)       → same assertions, fixtures, parametrize │
│  async/await (Week 1)  → endpoints are async; tests handle it  │
│  Pydantic (Week 3)     → validation errors become 422s         │
│  Depends() (Week 3)    → dependency_overrides is the payoff     │
│  Error handling (W1+3) → HTTPException becomes status codes     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                TESTING FASTAPI APPLICATIONS                     │
│                    (3-3.5 Hour Lecture)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (30 min)                                   │
│  ├─ 1.1 The Code Under Test                                     │
│  ├─ 1.2 The Naive Attempt (Demonstration)                       │
│  ├─ 1.3 What You're Actually Testing (The HTTP Contract)        │
│  └─ 1.4 The Vending Machine Analogy                             │
│                                                                 │
│  PART 2: THE TOOL                                               │
│  ├─ 2.1 Meet TestClient (The Testing Panel)                     │
│  ├─ 2.2 Your First API Test                                     │
│  ├─ 2.3 What TestClient Actually Does (Under the Hood)          │
│  ├─ 2.4 Every HTTP Method in TestClient          [MODIFIED]     │
│  ├─ 2.5 Building a Client Fixture (Connection to Week 2)        │
│  └─ 2.6 Cookies, File Uploads, Redirects,                       │
│          and Custom Response Headers             [NEW]          │
│                                                                 │
│  PART 3: TESTING WHAT MATTERS                                   │
│  ├─ 3.1 Testing Success Responses (The Happy Path)              │
│  ├─ 3.2 Testing Validation Errors (Pydantic Says No)            │
│  ├─ 3.3 Testing Not Found and Business Errors                   │
│  ├─ 3.4 Parametrize for Coverage (Connection to Week 2)         │
│  ├─ 3.5 Testing Response Shape (Trust, but Verify)              │
│  ├─ 3.6 Advanced Query Parameter Testing         [NEW]          │
│  ├─ 3.7 Testing Idempotency and Error Consistency [NEW]         │
│  └─ 3.8 Rate Limiting, Pagination, and OpenAPI   [NEW]          │
│                                                                 │
│  PART 4: ISOLATING YOUR CODE                                    │
│  ├─ 4.1 The Shared State Problem (Demonstration)                │
│  ├─ 4.2 dependency_overrides (FastAPI's Secret Weapon)          │
│  ├─ 4.3 Overriding Yield Dependencies                           │
│  ├─ 4.4 Unit vs Integration: When to Override, When Not To      │
│  ├─ 4.5 Overrides vs Mocks (Connection to Week 2)               │
│  ├─ 4.6 Testing Authenticated Endpoints          [NEW]          │
│  ├─ 4.7 Testing Middleware                       [NEW]          │
│  └─ 4.8 Mocking External HTTP Calls              [NEW]          │
│                                                                 │
│  PART 5: ASYNC TESTING & TEST ARCHITECTURE                      │
│  ├─ 5.1 When TestClient Isn't Enough                            │
│  ├─ 5.2 httpx.AsyncClient with ASGITransport     [MODIFIED]     │
│  ├─ 5.3 Test File Organization                                  │
│  ├─ 5.4 Factory Pattern for Test Data                           │
│  ├─ 5.5 Your Testing Strategy                                   │
│  ├─ 5.6 Testing Lifespan Events                  [NEW]          │
│  ├─ 5.7 Test Coverage Measurement                [NEW]          │
│  ├─ 5.8 Testing Background Tasks                 [NEW]          │
│  ├─ 5.9 Database Testing Preview (Bridge to Week 6) [NEW]       │
│  └─ 5.10 Debugging Failed Tests                  [NEW]          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Code Under Test

**Here's a simplified version of the Task Manager you're building. Keep it on screen — we'll refer to it constantly.**

```python
# app.py — Task Manager API (simplified from your Week 3 project)
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field

app = FastAPI()

# In-memory storage (no database yet — that's Week 5-6)
tasks_db: dict[int, dict] = {}
_id_counter: int = 0


class TaskCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    priority: int = Field(ge=1, le=5, default=3)


class TaskResponse(BaseModel):
    id: int
    title: str
    priority: int
    completed: bool


@app.post("/tasks", status_code=201)
async def create_task(task: TaskCreate) -> TaskResponse:
    global _id_counter
    _id_counter += 1
    task_data = {"id": _id_counter, **task.model_dump(), "completed": False}
    tasks_db[_id_counter] = task_data
    return TaskResponse(**task_data)


@app.get("/tasks/{task_id}")
async def get_task(task_id: int) -> TaskResponse:
    if task_id not in tasks_db:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")
    return TaskResponse(**tasks_db[task_id])


@app.get("/tasks")
async def list_tasks() -> list[TaskResponse]:
    return [TaskResponse(**t) for t in tasks_db.values()]
```

**Three endpoints. Simple CRUD. Now let's test them.**

---

## 1.2 The Naive Attempt

**In Week 2, you learned: call the function, check the result. Let's try that.**

```python
# test_naive.py — Attempt 1: treat it like a regular function
from app import create_task, TaskCreate

def test_create_task():
    task_input = TaskCreate(title="Write docs", priority=2)
    result = create_task(task_input)
    print(result)

test_create_task()
```

**Run it:**

```
<coroutine object create_task at 0x7f8b3c2a1e40>
RuntimeWarning: coroutine 'create_task' was never awaited
```

**Right — it's `async def`. You learned this in Week 1. Let's fix it.**

```python
# test_naive.py — Attempt 2: await the coroutine
import asyncio
from app import create_task, TaskCreate

async def test_create_task():
    task_input = TaskCreate(title="Write docs", priority=2)
    result = await create_task(task_input)
    
    print(type(result))     # <class 'app.TaskResponse'>
    print(result.title)     # "Write docs"
    print(result.priority)  # 2
    print(result.id)        # 1
    
    # Looks correct! Ship it!
    
    # ... but wait. One question:
    # What HTTP status code will the CLIENT receive?
    print(result.status_code)  # ❌ AttributeError!

asyncio.run(test_create_task())
```

**Now ask the class:**

> "The function returned a `TaskResponse` Python object. But your client doesn't receive Python objects. They receive HTTP responses with status codes, headers, and JSON bodies. Where is the status code? Where is the `Content-Type: application/json` header? Where is the 201 Created we specified in the decorator?"

Answer: **Nowhere. We bypassed the entire HTTP layer.**

---

**It gets worse. Try testing the error case:**

```python
# test_naive.py — Attempt 3: test a 404
import asyncio
from app import get_task

async def test_not_found():
    try:
        result = await get_task(task_id=999)
    except Exception as e:
        print(type(e))    # <class 'fastapi.exceptions.HTTPException'>
        print(e.status_code)  # 404
        print(e.detail)       # "Task 999 not found"
    
    # We caught a PYTHON EXCEPTION.
    # But clients don't catch Python exceptions.
    # Clients receive an HTTP response: {"detail": "Task 999 not found"}
    # with status code 404.
    #
    # If you have a custom exception handler (@app.exception_handler),
    # it was COMPLETELY BYPASSED.

asyncio.run(test_not_found())
```

**The insight:**

> "Calling the function directly is like testing a car engine on a workbench. You can verify the engine runs, but you have NO idea if the steering wheel turns the wheels, if the brake pedal stops the car, or if the speedometer shows the right number. You need to sit in the car and drive it."

---

## 1.3 What You're Actually Testing (The HTTP Contract)

**When a client talks to your API, this is the full pipeline:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE FULL REQUEST PIPELINE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT sends HTTP request                                      │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────┐                                                │
│  │ Middleware   │  ← Logging, CORS, timing                      │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Routing     │  ← Match URL to endpoint function              │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Path params │  ← Extract {task_id} from URL, coerce to int  │
│  │ + Query     │  ← Parse ?completed=true from query string     │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Pydantic    │  ← Validate JSON body → TaskCreate             │
│  │ Validation  │  ← REJECT invalid data with 422               │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Dependency  │  ← Resolve Depends() tree                      │
│  │ Injection   │                                                │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────────────────────────────────┐                    │
│  │  YOUR FUNCTION   ← You are testing HERE │                    │
│  └──────┬──────────────────────────────────┘                    │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Response    │  ← Serialize to JSON, set status code          │
│  │ Serializer  │  ← Apply response_model filtering              │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Exception   │  ← Convert HTTPException → proper response     │
│  │ Handlers    │                                                │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  CLIENT receives HTTP response                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When you call the function directly, you test ONE box. When you test through HTTP, you test ALL of them.**

```
┌─────────────────────────────────────────────────────────────────┐
│               DIRECT CALL vs HTTP TEST                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DIRECT CALL (await create_task(task)):                         │
│  ✅ Function logic                                               │
│  ❌ Middleware                                                   │
│  ❌ Routing                                                      │
│  ❌ Path/query parameter extraction                              │
│  ❌ Pydantic request validation                                  │
│  ❌ Dependency injection                                         │
│  ❌ Status code assignment                                       │
│  ❌ Response serialization                                       │
│  ❌ Exception handlers                                           │
│                                                                 │
│  HTTP TEST (client.post("/tasks", json={...})):                 │
│  ✅ Function logic                                               │
│  ✅ Middleware                                                    │
│  ✅ Routing                                                      │
│  ✅ Path/query parameter extraction                              │
│  ✅ Pydantic request validation                                  │
│  ✅ Dependency injection                                         │
│  ✅ Status code assignment                                       │
│  ✅ Response serialization                                       │
│  ✅ Exception handlers                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**This is the HTTP contract:**

> "The HTTP contract is the agreement between your API and its clients. It says: 'If you send THIS request, you will receive THIS response — with THIS status code, THIS body shape, and THIS error when something goes wrong.' Testing through HTTP tests the contract. Testing the function directly does not."

---

## 1.4 The Vending Machine Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE VENDING MACHINE ANALOGY                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your API is a VENDING MACHINE.                                 │
│                                                                 │
│  Customers interact with it through:                            │
│  ├─ BUTTONS (endpoints — /tasks, /tasks/{id})                   │
│  ├─ COIN SLOT (request body — the JSON payload)                 │
│  └─ DISPLAY (response — status code + JSON body)                │
│                                                                 │
│                                                                 │
│  TESTING THE FUNCTION DIRECTLY:                                 │
│  ──────────────────────────────                                 │
│  You open the back panel, reach in, and grab a snack.           │
│  It tells you the motor works. It tells you NOTHING about       │
│  whether a customer pressing B3 with $2.00 gets their chips.    │
│                                                                 │
│                                                                 │
│  TESTING THROUGH HTTP:                                          │
│  ─────────────────────                                          │
│  You walk up to the front, insert coins, press a button,        │
│  and check: Did the right snack drop? Did the display           │
│  show the right message? Did you get correct change?            │
│                                                                 │
│  THAT is testing the contract.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to testing:**

```
Vending Machine             │  FastAPI Testing
────────────────────────────│──────────────────────────────
The machine itself          │  Your FastAPI app
Buttons (A1, B3, C2)       │  Endpoints (POST /tasks)
Inserting coins             │  Sending a request body (JSON)
Display message             │  HTTP status code (201, 404, 422)
What drops into the slot    │  Response JSON body
Testing panel on the back   │  TestClient
Loading test products       │  Setting up test data / fixtures
Replacing the coin reader   │  dependency_overrides
  with a test one           │
```

---

# PART 2: THE TOOL

## 2.1 Meet TestClient (The Testing Panel)

**FastAPI provides `TestClient` — a way to send HTTP requests to your app WITHOUT starting a real server.**

```python
from fastapi.testclient import TestClient
from app import app

client = TestClient(app)
```

**That's it. One import, one line, and you have a fully functional HTTP client that talks directly to your app in-process.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    HOW TestClient WORKS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT TestClient (manual testing):                           │
│                                                                 │
│    Terminal 1:              Terminal 2:                          │
│    uvicorn app:app          curl http://localhost:8000/tasks     │
│    ↑ Real server running    ↑ Real HTTP over network            │
│                                                                 │
│                                                                 │
│  WITH TestClient (automated testing):                           │
│                                                                 │
│    Your test file:                                              │
│    client = TestClient(app)                                     │
│    client.get("/tasks")                                         │
│    ↑ No server. No network. Direct in-process call.             │
│      FastAPI's FULL pipeline runs, but no TCP/IP overhead.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In the vending machine analogy:**

> "TestClient is the testing panel built into the back of the machine. It lets you simulate button presses and coin insertions without being a real customer. But the machine's entire mechanism still runs — the motor turns, the belt moves, the snack drops. You're testing the real machine, just without the real customer."

---

## 2.2 Your First API Test

**Let's write a real test. Finally.**

```python
# test_tasks.py
from fastapi.testclient import TestClient
from app import app

client = TestClient(app)


def test_create_task():
    response = client.post(
        "/tasks",
        json={"title": "Write documentation", "priority": 2}
    )
    
    # Check the HTTP status code — we couldn't do this before!
    assert response.status_code == 201
    
    # Check the response body
    data = response.json()
    assert data["title"] == "Write documentation"
    assert data["priority"] == 2
    assert data["completed"] is False
    assert "id" in data
```

**Run it:**

```
$ pytest test_tasks.py -v

test_tasks.py::test_create_task PASSED
```

**Now ask the class:**

> "Compare this to the naive attempt in Part 1. What are we testing now that we weren't before?"

Answer:

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT WE'RE ACTUALLY TESTING NOW                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ The URL "/tasks" is correctly routed to create_task          │
│  ✅ The JSON body is parsed and validated by Pydantic            │
│  ✅ The status code is 201 (not default 200)                     │
│  ✅ The response body is serialized to JSON correctly            │
│  ✅ The response matches the TaskResponse schema                 │
│  ✅ The function logic creates the task with correct fields      │
│                                                                 │
│  All from ONE test. All impossible with a direct function call. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 What TestClient Actually Does (Under the Hood)

**You don't need to understand every detail, but the mental model matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                TestClient UNDER THE HOOD                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  client.post("/tasks", json={...})                              │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────┐                                       │
│  │  Build HTTP Request  │  Construct a real HTTP request object │
│  │  (method, URL, body, │  with proper headers, encoding, etc. │
│  │   headers)           │                                       │
│  └──────────┬───────────┘                                       │
│             ▼                                                    │
│  ┌──────────────────────┐                                       │
│  │  Send to ASGI app    │  Call your FastAPI app directly       │
│  │  (no network!)       │  through ASGI protocol (in-process)  │
│  └──────────┬───────────┘                                       │
│             ▼                                                    │
│  ┌──────────────────────┐                                       │
│  │  FastAPI processes   │  FULL pipeline: routing, validation, │
│  │  the request         │  dependencies, your function,        │
│  │                      │  serialization, exception handlers   │
│  └──────────┬───────────┘                                       │
│             ▼                                                    │
│  ┌──────────────────────┐                                       │
│  │  Return Response     │  Real HTTP response object with      │
│  │  object              │  status_code, headers, body          │
│  └──────────────────────┘                                       │
│                                                                 │
│  KEY POINT: TestClient is SYNCHRONOUS.                          │
│  It handles the async event loop internally.                    │
│  Your test functions are plain def, not async def.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why synchronous? Connection to Week 1 Lecture 3:**

> "Remember from the Async lecture: `asyncio.run()` starts the event loop. TestClient does that for you under the hood. You write normal `def test_...()` functions and let TestClient manage the async machinery. This is simpler for most tests — we'll cover async testing in Part 5 for when you need more control."

---

## 2.4 Every HTTP Method in TestClient

**TestClient mirrors the HTTP methods you learned in Week 3 Lecture 1:**

```python
from fastapi.testclient import TestClient
from app import app

client = TestClient(app)

# ── GET (retrieve) ─────────────────────────────────────
response = client.get("/tasks")
response = client.get("/tasks/42")
response = client.get("/tasks", params={"completed": "true"})

# ── POST (create) ──────────────────────────────────────
response = client.post("/tasks", json={"title": "New task"})

# ── PUT (full replace) ─────────────────────────────────
response = client.put("/tasks/42", json={"title": "Updated", "priority": 5})

# ── PATCH (partial update) ─────────────────────────────
response = client.patch("/tasks/42", json={"priority": 1})

# ── DELETE (remove) ────────────────────────────────────
response = client.delete("/tasks/42")
```

**Sending different types of data:**

```python
# JSON body (most common for APIs)
response = client.post("/tasks", json={"title": "Test"})

# Query parameters
response = client.get("/tasks", params={"completed": "true", "priority": "3"})

# Custom headers
response = client.get(
    "/tasks",
    headers={"X-Request-ID": "abc-123"}
)

# Path parameters — just build the URL string
task_id = 42
response = client.get(f"/tasks/{task_id}")
```

```python
# ── Form data (HTML form submissions, OAuth2 password grant) ───
response = client.post(
    "/auth/token",
    data={"username": "alice", "password": "secret"}
    # IMPORTANT: data={} sends application/x-www-form-urlencoded
    # This is NOT the same as json={}. Different Content-Type entirely.
)

# ── Single file upload (multipart/form-data) ───────────────────
response = client.post(
    "/upload",
    files={"file": ("report.pdf", b"PDF bytes here", "application/pdf")}
    #              ^ filename      ^ raw bytes         ^ MIME type
)

# ── File AND form fields in the same request ───────────────────
# (most real upload endpoints need both)
response = client.post(
    "/profile/photo",
    files={"photo": ("avatar.png", b"PNG bytes here", "image/png")},
    data={"alt_text": "My profile picture"}   # form field alongside file
)

# ── Multiple files under the same field name ───────────────────
response = client.post(
    "/upload/batch",
    files=[
        ("files", ("a.txt", b"content a", "text/plain")),
        ("files", ("b.txt", b"content b", "text/plain")),
    ]
)
```

```
┌─────────────────────────────────────────────────────────────────┐
│           json= vs data= vs files= — WHAT EACH SENDS           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  json={"key": "val"}                                            │
│  ── Content-Type: application/json                              │
│  ── Body: {"key": "val"}  (serialised JSON)                     │
│  ── Use for: REST APIs, Pydantic request bodies                 │
│                                                                 │
│  data={"key": "val"}                                            │
│  ── Content-Type: application/x-www-form-urlencoded             │
│  ── Body: key=val  (URL-encoded string)                         │
│  ── Use for: HTML forms, OAuth2 password flow                   │
│                                                                 │
│  files={"file": ("name", bytes, "mime")}                        │
│  ── Content-Type: multipart/form-data; boundary=...             │
│  ── Body: binary parts separated by a boundary string           │
│  ── Use for: file uploads                                       │
│                                                                 │
│  files + data together                                          │
│  ── Content-Type: multipart/form-data                           │
│  ── Body: file part + field parts in one multipart request      │
│  ── Use for: file upload with accompanying metadata             │
│                                                                 │
│  COMMON MISTAKE: sending a file with json={} instead of         │
│  files={}. You'll get 422 — Pydantic can't parse a JSON body   │
│  as an UploadFile. FastAPI knows the difference.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Reading the response:**

```python
response = client.post("/tasks", json={"title": "Test"})

# Status code (int)
response.status_code     # 201

# Body as JSON (dict or list)
response.json()          # {"id": 1, "title": "Test", ...}

# Body as raw text
response.text            # '{"id":1,"title":"Test",...}'

# Response headers
response.headers         # Headers({...})
response.headers["content-type"]  # "application/json"
```

---

## 2.5 Building a Client Fixture (Connection to Week 2)

**Connection to what you've learned:**

> "In Week 2, you learned `@pytest.fixture` for test setup. Let's use that to create a reusable client."

```python
# test_tasks.py
import pytest
from fastapi.testclient import TestClient
from app import app


@pytest.fixture
def client() -> TestClient:
    """Create a fresh TestClient for each test."""
    return TestClient(app)


def test_create_task(client: TestClient):
    response = client.post("/tasks", json={"title": "Test"})
    assert response.status_code == 201


def test_get_task(client: TestClient):
    # First create a task
    create_response = client.post("/tasks", json={"title": "Test"})
    task_id = create_response.json()["id"]
    
    # Then retrieve it
    response = client.get(f"/tasks/{task_id}")
    assert response.status_code == 200
    assert response.json()["title"] == "Test"
```

**Using TestClient as a context manager (connection to Week 1 Lecture 2):**

> "Remember context managers from Week 1? TestClient supports the `with` statement. This triggers FastAPI's startup and shutdown events (called 'lifespan'). For your current project this doesn't matter, but it will when your app has initialization logic."

```python
@pytest.fixture
def client() -> TestClient:
    # Triggers lifespan events (startup/shutdown)
    with TestClient(app) as c:
        yield c
```

```
┌─────────────────────────────────────────────────────────────────┐
│         TestClient: DIRECT vs CONTEXT MANAGER                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TestClient(app)                                                │
│  ── Simple instantiation                                        │
│  ── Does NOT trigger lifespan events                            │
│  ── Fine for apps with no startup/shutdown logic                │
│  ── ✅ Use this for your Week 3-4 project                       │
│                                                                 │
│  with TestClient(app) as client:                                │
│  ── Context manager form                                        │
│  ── DOES trigger lifespan events (startup → yield → shutdown)   │
│  ── Required when your app initializes resources on startup     │
│  ── ✅ Use this when you add database connections (Week 6)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.6 Cookies, File Uploads, Redirects, and Custom Response Headers

### Cookies

**Many APIs use cookies for sessions, refresh tokens, or CSRF protection. TestClient handles cookies exactly like a browser would.**

```python
# ── Sending cookies with a request ────────────────────────────
response = client.get(
    "/profile",
    cookies={"session_id": "abc-123"}
)

# ── Reading cookies the server sets on the response ───────────
response = client.post(
    "/auth/login",
    json={"username": "alice", "password": "secret"}
)
assert response.status_code == 200
assert "refresh_token" in response.cookies
assert response.cookies["refresh_token"] != ""

# ── Cookie jar: TestClient persists cookies between requests ──
# (when reusing the same client instance inside a with block)
with TestClient(app) as sticky_client:
    login_resp = sticky_client.post(
        "/auth/login",
        json={"username": "alice", "password": "secret"}
    )
    # The client now has the session cookie stored automatically
    profile_resp = sticky_client.get("/profile")  # Cookie sent automatically
    assert profile_resp.status_code == 200
```

**Testing that the server correctly sets a cookie:**

```python
# app.py extension:
from fastapi.responses import JSONResponse

@app.post("/auth/login")
async def login(credentials: LoginRequest):
    # validate credentials...
    response = JSONResponse(content={"message": "Logged in"})
    response.set_cookie(
        key="session_id",
        value="generated-session-token",
        httponly=True,
        secure=True,
        samesite="lax",
    )
    return response

# test:
def test_login_sets_session_cookie(client: TestClient):
    response = client.post(
        "/auth/login",
        json={"username": "alice", "password": "correct"}
    )
    assert response.status_code == 200

    # Verify the cookie exists and has the right security flags
    assert "session_id" in response.cookies

    # Inspect the raw Set-Cookie header for security attributes
    set_cookie_header = response.headers.get("set-cookie", "")
    assert "HttpOnly" in set_cookie_header
    assert "SameSite=lax" in set_cookie_header
```

---

### File Uploads and Downloads

**Extend the app with a simple upload endpoint:**

```python
# app.py extension
from fastapi import UploadFile, File

@app.post("/upload", status_code=201)
async def upload_file(file: UploadFile = File(...)) -> dict:
    contents = await file.read()
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents),
    }

@app.post("/upload/image", status_code=201)
async def upload_image(
    image: UploadFile = File(...),
    alt_text: str = Form(default=""),
) -> dict:
    if not image.content_type.startswith("image/"):
        raise HTTPException(
            status_code=400,
            detail=f"Expected an image, got '{image.content_type}'"
        )
    contents = await image.read()
    return {
        "filename": image.filename,
        "size": len(contents),
        "alt_text": alt_text,
    }
```

**Testing file upload endpoints:**

```python
def test_upload_file_returns_metadata(client: TestClient):
    """Uploaded file metadata should appear in the response."""
    response = client.post(
        "/upload",
        files={"file": ("report.pdf", b"PDF content here", "application/pdf")}
    )
    assert response.status_code == 201
    data = response.json()
    assert data["filename"] == "report.pdf"
    assert data["content_type"] == "application/pdf"
    assert data["size"] == len(b"PDF content here")


def test_upload_image_with_alt_text(client: TestClient):
    """Image upload with form field alongside file."""
    response = client.post(
        "/upload/image",
        files={"image": ("cat.png", b"\x89PNG...", "image/png")},
        data={"alt_text": "A cute cat"}
    )
    assert response.status_code == 201
    assert response.json()["alt_text"] == "A cute cat"


def test_upload_image_rejects_non_images(client: TestClient):
    """Non-image file should return 400 Bad Request."""
    response = client.post(
        "/upload/image",
        files={"image": ("data.csv", b"a,b,c", "text/csv")}
    )
    assert response.status_code == 400
    assert "image" in response.json()["detail"].lower()


def test_upload_empty_file(client: TestClient):
    """Empty file upload — document your chosen behavior."""
    response = client.post(
        "/upload",
        files={"file": ("empty.txt", b"", "text/plain")}
    )
    # Does your API accept empty files? 201 or 400?
    # This test DOCUMENTS that decision.
    assert response.status_code == 201
    assert response.json()["size"] == 0
```

**Testing file download endpoints:**

```python
# app.py extension
from fastapi.responses import StreamingResponse
import io

@app.get("/tasks/{task_id}/export")
async def export_task(task_id: int) -> StreamingResponse:
    task = tasks_db.get(task_id)
    if not task:
        raise HTTPException(status_code=404, detail="Task not found")
    csv_content = f"id,title,priority\n{task['id']},{task['title']},{task['priority']}\n"
    return StreamingResponse(
        io.StringIO(csv_content),
        media_type="text/csv",
        headers={"Content-Disposition": f"attachment; filename=task_{task_id}.csv"}
    )

# test:
def test_export_task_returns_csv(client: TestClient):
    """Export endpoint should return a downloadable CSV file."""
    create_resp = client.post("/tasks", json={"title": "Exported task", "priority": 2})
    task_id = create_resp.json()["id"]

    response = client.get(f"/tasks/{task_id}/export")

    assert response.status_code == 200
    assert response.headers["content-type"] == "text/csv; charset=utf-8"
    assert "attachment" in response.headers["content-disposition"]
    assert f"task_{task_id}.csv" in response.headers["content-disposition"]

    # Check the CSV body contains the right data
    lines = response.text.strip().split("\n")
    assert lines[0] == "id,title,priority"
    assert "Exported task" in lines[1]
```

---

### Redirects

**FastAPI uses `RedirectResponse` to send 307/308 status codes. TestClient follows redirects by default — you must disable that to test the redirect itself.**

```python
# app.py extension
from fastapi.responses import RedirectResponse

@app.get("/tasks/latest")
async def get_latest_task():
    """Redirect to the most recently created task."""
    if not tasks_db:
        raise HTTPException(status_code=404, detail="No tasks exist")
    latest_id = max(tasks_db.keys())
    return RedirectResponse(url=f"/tasks/{latest_id}", status_code=307)
```

```python
def test_redirect_status_code(client: TestClient):
    """GET /tasks/latest should redirect, not return data directly."""
    client.post("/tasks", json={"title": "Only task"})

    # follow_redirects=False: stop at the redirect, don't follow it
    response = client.get("/tasks/latest", follow_redirects=False)

    assert response.status_code == 307
    assert response.headers["location"] == "/tasks/1"


def test_redirect_is_followed_by_default(client: TestClient):
    """Without follow_redirects=False, TestClient follows the redirect."""
    client.post("/tasks", json={"title": "Target task"})

    response = client.get("/tasks/latest")   # follows redirect by default

    assert response.status_code == 200       # final destination's status
    assert response.json()["title"] == "Target task"


def test_redirect_on_empty_db_returns_404(client: TestClient):
    """If no tasks exist, the redirect endpoint should return 404."""
    response = client.get("/tasks/latest", follow_redirects=False)
    assert response.status_code == 404
```

```
┌─────────────────────────────────────────────────────────────────┐
│              REDIRECT STATUS CODES — QUICK GUIDE                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  301 Moved Permanently — method may change to GET on redirect   │
│  302 Found            — same; method may change                 │
│  303 See Other        — explicitly tells client to use GET      │
│  307 Temporary        — method PRESERVED (POST stays POST)      │
│  308 Permanent        — method PRESERVED (POST stays POST)      │
│                                                                 │
│  For API redirects: prefer 307 (preserve method) or 308.       │
│  For form redirects after POST: use 303 (redirect to GET).     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Custom Response Headers

**Testing that your app sets the right response headers:**

```python
def test_post_response_includes_location_header(client: TestClient):
    """REST convention: POST that creates a resource should include Location."""
    response = client.post("/tasks", json={"title": "New task"})
    assert response.status_code == 201
    # If your app follows REST conventions:
    # assert "location" in response.headers
    # assert response.headers["location"].startswith("/tasks/")

def test_content_type_is_json(client: TestClient):
    """All endpoints should return application/json Content-Type."""
    response = client.get("/tasks")
    assert "application/json" in response.headers["content-type"]
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your endpoint POST /documents accepts a PDF file (field name   │
│  "document") and an optional form field "title" (string).       │
│  Write THREE tests: (1) a successful upload with title,         │
│  (2) a rejection when a non-PDF is uploaded, and (3) verify     │
│  the response contains the filename and size.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION

def test_pdf_upload_with_title(client: TestClient):
    response = client.post(
        "/documents",
        files={"document": ("report.pdf", b"%PDF-1.4 ...", "application/pdf")},
        data={"title": "Q4 Report"}
    )
    assert response.status_code == 201
    assert response.json()["filename"] == "report.pdf"
    assert response.json()["title"] == "Q4 Report"


def test_non_pdf_upload_rejected(client: TestClient):
    response = client.post(
        "/documents",
        files={"document": ("image.png", b"\x89PNG...", "image/png")}
    )
    assert response.status_code == 400
    assert "pdf" in response.json()["detail"].lower()


def test_upload_response_contains_filename_and_size(client: TestClient):
    content = b"Hello PDF"
    response = client.post(
        "/documents",
        files={"document": ("doc.pdf", content, "application/pdf")}
    )
    assert response.status_code == 201
    data = response.json()
    assert "filename" in data
    assert data["size"] == len(content)
```

---

# PART 3: TESTING WHAT MATTERS

## 3.1 Testing Success Responses (The Happy Path)

**The happy path is the scenario where everything goes right. Start here, but don't stop here.**

```python
def test_create_task_returns_201(client: TestClient):
    """POST /tasks with valid data should return 201 Created."""
    response = client.post(
        "/tasks",
        json={"title": "Learn testing", "priority": 4}
    )
    
    assert response.status_code == 201  # Not 200 — the specific code matters
    
    data = response.json()
    assert data["title"] == "Learn testing"
    assert data["priority"] == 4
    assert data["completed"] is False  # Default value
    assert isinstance(data["id"], int)  # ID was assigned


def test_create_task_with_defaults(client: TestClient):
    """POST /tasks with only required fields uses defaults."""
    response = client.post("/tasks", json={"title": "Minimal task"})
    
    assert response.status_code == 201
    assert response.json()["priority"] == 3  # Default from Pydantic model


def test_get_existing_task(client: TestClient):
    """GET /tasks/{id} for an existing task returns 200."""
    # Arrange: create a task first
    create_resp = client.post("/tasks", json={"title": "Find me"})
    task_id = create_resp.json()["id"]
    
    # Act: retrieve it
    response = client.get(f"/tasks/{task_id}")
    
    # Assert
    assert response.status_code == 200
    assert response.json()["title"] == "Find me"
    assert response.json()["id"] == task_id
```

**Notice the Arrange-Act-Assert pattern (from Week 2):**

```
┌─────────────────────────────────────────────────────────────────┐
│                   ARRANGE-ACT-ASSERT                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ARRANGE: Set up the preconditions                              │
│  ── Create a task so we have something to GET                   │
│                                                                 │
│  ACT: Perform the action being tested                           │
│  ── Send the GET request                                        │
│                                                                 │
│  ASSERT: Check the outcome                                      │
│  ── Status code is 200                                          │
│  ── Response body matches expectations                          │
│                                                                 │
│  This is the same pattern from Week 2,                          │
│  applied to HTTP requests instead of function calls.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Testing Validation Errors (Pydantic Says No)

**Connection to Week 3 Lecture 3:**

> "Remember Pydantic's field constraints? `Field(min_length=1, max_length=100)` and `Field(ge=1, le=5)` in your TaskCreate model. When a client sends data that violates those constraints, FastAPI returns 422 Unprocessable Entity — automatically. You MUST test this."

```python
def test_create_task_empty_title_returns_422(client: TestClient):
    """POST /tasks with empty title should be rejected."""
    response = client.post("/tasks", json={"title": "", "priority": 3})
    
    assert response.status_code == 422  # Unprocessable Entity
```

**But what does the 422 response BODY look like?**

```python
def test_create_task_empty_title_error_detail(client: TestClient):
    """422 response should include structured error details."""
    response = client.post("/tasks", json={"title": ""})
    
    assert response.status_code == 422
    
    # FastAPI returns a structured error body:
    error_body = response.json()
    print(error_body)
```

**Output:**

```json
{
    "detail": [
        {
            "type": "string_too_short",
            "loc": ["body", "title"],
            "msg": "String should have at least 1 character",
            "input": "",
            "ctx": {"min_length": 1}
        }
    ]
}
```

**Anatomy of a 422 error:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   422 ERROR ANATOMY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  {                                                              │
│    "detail": [                 ← List of errors (can be many)   │
│      {                                                          │
│        "type": "string_too_short",  ← Machine-readable type    │
│        "loc": ["body", "title"],    ← WHERE the error is       │
│                  ▲       ▲                                      │
│                  │       └── Field name                         │
│                  └────────── In the request body                │
│        "msg": "String should...",   ← Human-readable message   │
│        "input": "",                 ← What they actually sent   │
│        "ctx": {"min_length": 1}     ← Constraint that failed   │
│      }                                                          │
│    ]                                                            │
│  }                                                              │
│                                                                 │
│  "loc" values: ["body", "field"]    ← JSON body field           │
│                ["path", "task_id"]  ← Path parameter            │
│                ["query", "page"]    ← Query parameter           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Asserting specific error details:**

```python
def test_create_task_empty_title_error_location(client: TestClient):
    """Error should point to the correct field."""
    response = client.post("/tasks", json={"title": ""})
    
    errors = response.json()["detail"]
    assert len(errors) == 1
    assert errors[0]["loc"] == ["body", "title"]
    assert errors[0]["type"] == "string_too_short"
```

**Testing path parameter coercion:**

```python
def test_get_task_invalid_id_type(client: TestClient):
    """GET /tasks/abc should return 422 — 'abc' is not an int."""
    response = client.get("/tasks/abc")
    
    assert response.status_code == 422
    errors = response.json()["detail"]
    assert errors[0]["loc"] == ["path", "task_id"]
```

> "FastAPI declared `task_id: int` in the path. When a client sends `/tasks/abc`, FastAPI can't coerce `'abc'` to an integer and returns 422. This validation is part of the HTTP contract — test it."

---

## 3.3 Testing Not Found and Business Errors

**These test the error paths your application defines explicitly.**

```python
def test_get_nonexistent_task_returns_404(client: TestClient):
    """GET /tasks/{id} for a missing task should return 404."""
    response = client.get("/tasks/99999")
    
    assert response.status_code == 404
    assert response.json()["detail"] == "Task 99999 not found"


def test_404_response_structure(client: TestClient):
    """404 response should have the standard error format."""
    response = client.get("/tasks/99999")
    
    assert response.status_code == 404
    data = response.json()
    assert "detail" in data
    assert isinstance(data["detail"], str)
```

**Connection to Week 3 Lecture 4 — Custom exception handlers:**

> "If your app defines a custom exception handler with `@app.exception_handler`, testing through TestClient is the ONLY way to verify it works. A direct function call would raise the raw exception and skip your handler entirely."

```python
# In your app (from Week 3 Lecture 4):
class TaskNotFoundError(Exception):
    def __init__(self, task_id: int):
        self.task_id = task_id

@app.exception_handler(TaskNotFoundError)
async def task_not_found_handler(request, exc: TaskNotFoundError):
    return JSONResponse(
        status_code=404,
        content={
            "error": "not_found",
            "message": f"Task {exc.task_id} does not exist",
        }
    )

# Test that the custom handler formats the error correctly:
def test_custom_not_found_format(client: TestClient):
    response = client.get("/tasks/99999")
    
    assert response.status_code == 404
    data = response.json()
    assert data["error"] == "not_found"      # Custom field
    assert "99999" in data["message"]         # Includes the ID
```

---

## 3.4 Parametrize for Coverage (Connection to Week 2)

**Connection to what you've learned:**

> "In Week 2, you learned `@pytest.mark.parametrize` for running the same test with multiple inputs. This is PERFECT for validation testing — one test function, many invalid inputs."

```python
import pytest
from fastapi.testclient import TestClient
from app import app


@pytest.fixture
def client() -> TestClient:
    return TestClient(app)


@pytest.mark.parametrize("payload,expected_status", [
    # ── Happy path ──
    ({"title": "Valid task"}, 201),
    ({"title": "Valid", "priority": 1}, 201),
    ({"title": "Valid", "priority": 5}, 201),
    
    # ── Validation failures (422) ──
    ({"title": ""}, 422),                    # Too short (min_length=1)
    ({"title": "x" * 101}, 422),             # Too long (max_length=100)
    ({"title": "Valid", "priority": 0}, 422), # Priority too low (ge=1)
    ({"title": "Valid", "priority": 6}, 422), # Priority too high (le=5)
    ({}, 422),                                # Missing required field
])
def test_create_task_validation(
    client: TestClient,
    payload: dict,
    expected_status: int,
):
    response = client.post("/tasks", json=payload)
    assert response.status_code == expected_status
```

**Run it:**

```
$ pytest test_tasks.py::test_create_task_validation -v

test_create_task_validation[payload0-201] PASSED
test_create_task_validation[payload1-201] PASSED
test_create_task_validation[payload2-201] PASSED
test_create_task_validation[payload3-422] PASSED
test_create_task_validation[payload4-422] PASSED
test_create_task_validation[payload5-422] PASSED
test_create_task_validation[payload6-422] PASSED
test_create_task_validation[payload7-422] PASSED

8 passed
```

**8 scenarios. 1 test function. Every boundary of your Pydantic constraints verified.**

> "In the vending machine analogy: you're testing every coin denomination — valid coins, slugs, foreign coins, no coins. One testing procedure, many inputs."

**Make test IDs readable with `pytest.param`:**

```python
@pytest.mark.parametrize("payload,expected_status", [
    pytest.param(
        {"title": "Valid task"}, 201,
        id="valid-task-returns-201"
    ),
    pytest.param(
        {"title": ""}, 422,
        id="empty-title-rejected"
    ),
    pytest.param(
        {"title": "x" * 101}, 422,
        id="title-over-100-chars-rejected"
    ),
    pytest.param(
        {"title": "Valid", "priority": 0}, 422,
        id="priority-below-1-rejected"
    ),
])
def test_create_task_validation(client, payload, expected_status):
    response = client.post("/tasks", json=payload)
    assert response.status_code == expected_status
```

```
$ pytest -v

test_create_task_validation[valid-task-returns-201] PASSED
test_create_task_validation[empty-title-rejected] PASSED
test_create_task_validation[title-over-100-chars-rejected] PASSED
test_create_task_validation[priority-below-1-rejected] PASSED
```

---

## 3.5 Testing Response Shape (Trust, but Verify)

**It's not enough that the response returns 200. The body must have the right SHAPE.**

```python
def test_create_task_response_has_all_fields(client: TestClient):
    """Response must contain exactly the fields defined in TaskResponse."""
    response = client.post(
        "/tasks",
        json={"title": "Shape test", "priority": 2}
    )
    data = response.json()
    
    # Check ALL expected fields exist
    expected_fields = {"id", "title", "priority", "completed"}
    assert set(data.keys()) == expected_fields
    
    # Check field types
    assert isinstance(data["id"], int)
    assert isinstance(data["title"], str)
    assert isinstance(data["priority"], int)
    assert isinstance(data["completed"], bool)


def test_list_tasks_returns_array(client: TestClient):
    """GET /tasks should always return a JSON array."""
    response = client.get("/tasks")
    
    assert response.status_code == 200
    assert isinstance(response.json(), list)
```

**Why test the shape? Because clients depend on it.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHY TEST RESPONSE SHAPE?                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your frontend or mobile app expects:                           │
│  {                                                              │
│    "id": <int>,                                                 │
│    "title": <string>,                                           │
│    "priority": <int>,                                           │
│    "completed": <boolean>                                       │
│  }                                                              │
│                                                                 │
│  If you accidentally add, remove, or rename a field,            │
│  the frontend BREAKS. Your test catches this BEFORE deploy.     │
│                                                                 │
│  This is why response_model exists in FastAPI —                 │
│  but testing CONFIRMS it does what you think.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.6 Advanced Query Parameter Testing

**The one-line example in 2.4 showed how to send query parameters. This section is about how to ASSERT their behavior — required vs optional, type coercion, combinations, and edge cases.**

**Extend the app with a filterable list endpoint:**

```python
# app.py extension
from fastapi import Query

@app.get("/tasks")
async def list_tasks(
    completed: bool | None = None,           # Optional boolean filter
    priority: int | None = Query(            # Optional int, constrained
        default=None, ge=1, le=5
    ),
    search: str | None = None,               # Optional text filter
    size: int = Query(default=20, le=100),   # Capped at 100
    storage: TaskStorage = Depends(get_storage),
) -> list[TaskResponse]:
    tasks = storage.list_all()
    if completed is not None:
        tasks = [t for t in tasks if t["completed"] == completed]
    if priority is not None:
        tasks = [t for t in tasks if t["priority"] == priority]
    if search:
        tasks = [t for t in tasks if search.lower() in t["title"].lower()]
    return [TaskResponse(**t) for t in tasks[:size]]
```

**Testing optional vs required parameters:**

```python
def test_list_tasks_with_no_params_returns_all(client: TestClient):
    """All query params are optional — omitting them returns everything."""
    client.post("/tasks", json={"title": "Task A", "priority": 1})
    client.post("/tasks", json={"title": "Task B", "priority": 5})

    response = client.get("/tasks")   # No params at all
    assert response.status_code == 200
    assert len(response.json()) == 2


def test_filter_by_completed_true(client: TestClient):
    """completed=true should return only completed tasks."""
    client.post("/tasks", json={"title": "Done task"})
    # Assume PATCH to mark it complete (or pre-populate via storage fixture)
    response = client.get("/tasks", params={"completed": "true"})
    assert response.status_code == 200
    assert all(t["completed"] is True for t in response.json())


def test_filter_by_completed_false(client: TestClient):
    """completed=false should return only incomplete tasks."""
    client.post("/tasks", json={"title": "Pending task"})
    response = client.get("/tasks", params={"completed": "false"})
    assert all(t["completed"] is False for t in response.json())


def test_filter_by_priority(client: TestClient):
    """priority filter should only return matching tasks."""
    client.post("/tasks", json={"title": "Urgent", "priority": 5})
    client.post("/tasks", json={"title": "Low prio", "priority": 1})

    response = client.get("/tasks", params={"priority": "5"})
    data = response.json()
    assert len(data) == 1
    assert data[0]["priority"] == 5
```

**Testing query parameter type coercion — FastAPI coerces automatically, and REJECTS wrong types:**

```python
def test_invalid_priority_type_returns_422(client: TestClient):
    """priority=high cannot be coerced to int → 422."""
    response = client.get("/tasks", params={"priority": "high"})
    assert response.status_code == 422
    errors = response.json()["detail"]
    # Error should point to the query parameter, not the body
    assert any(e["loc"] == ["query", "priority"] for e in errors)


def test_priority_above_max_returns_422(client: TestClient):
    """priority=6 violates ge=1, le=5 → 422."""
    response = client.get("/tasks", params={"priority": "6"})
    assert response.status_code == 422


def test_size_above_limit_returns_422(client: TestClient):
    """size has le=100. Sending 101 → 422."""
    response = client.get("/tasks", params={"size": "101"})
    assert response.status_code == 422
    errors = response.json()["detail"]
    assert any(e["loc"] == ["query", "size"] for e in errors)
```

**Testing filter combinations (AND logic):**

```python
def test_combined_filters_are_anded(client: TestClient):
    """Multiple filters narrow results — they combine with AND."""
    client.post("/tasks", json={"title": "Match both", "priority": 5})
    client.post("/tasks", json={"title": "Priority only", "priority": 5})
    # "Match both" would be marked complete in a real test
    # This demonstrates the pattern — adapt to your storage

    response = client.get(
        "/tasks",
        params={"priority": "5"}
    )
    data = response.json()
    # All returned tasks match the priority filter
    assert all(t["priority"] == 5 for t in data)


def test_search_is_case_insensitive(client: TestClient):
    """Text search should match regardless of case."""
    client.post("/tasks", json={"title": "Write Documentation"})

    response = client.get("/tasks", params={"search": "documentation"})
    assert response.status_code == 200
    assert len(response.json()) == 1
    assert "Documentation" in response.json()[0]["title"]


def test_search_no_match_returns_empty_list(client: TestClient):
    """Search with no matches should return [] not 404."""
    client.post("/tasks", json={"title": "Existing task"})

    response = client.get("/tasks", params={"search": "xyzzy"})
    assert response.status_code == 200
    assert response.json() == []
```

> "Notice: no match → `[]` with status 200. Never return 404 for an empty collection. A collection that happens to be empty is still a valid, successful response. 404 means the COLLECTION ITSELF doesn't exist."

---

## 3.7 Testing Idempotency and Error Response Consistency

### Testing Idempotency

**The HTTP spec defines idempotency: calling the same operation multiple times produces the same result as calling it once. PUT and DELETE must be idempotent. Test it — don't assume it.**

```python
def test_delete_is_idempotent(client: TestClient):
    """Deleting the same resource twice should produce consistent behavior."""
    create_resp = client.post("/tasks", json={"title": "Delete me"})
    task_id = create_resp.json()["id"]

    first_delete = client.delete(f"/tasks/{task_id}")
    assert first_delete.status_code == 204   # Success: deleted

    second_delete = client.delete(f"/tasks/{task_id}")
    assert second_delete.status_code == 404  # Resource is gone — consistent 404


def test_put_is_idempotent(client: TestClient):
    """Calling PUT twice with the same payload should yield identical results."""
    create_resp = client.post("/tasks", json={"title": "Original", "priority": 3})
    task_id = create_resp.json()["id"]

    update_payload = {"title": "Updated Title", "priority": 5, "completed": False}

    first_put = client.put(f"/tasks/{task_id}", json=update_payload)
    second_put = client.put(f"/tasks/{task_id}", json=update_payload)

    assert first_put.status_code == 200
    assert second_put.status_code == 200
    assert first_put.json() == second_put.json()   # Identical results
```

> "Why does DELETE return 404 on the second call? Some argue it should return 204 again — the outcome is the same: the resource doesn't exist. Both are valid. What matters is that you DOCUMENT the behavior with a test. The test is your spec."

---

### Testing Error Response Consistency

**All your error responses should follow the same format. Clients parse your errors programmatically — inconsistent formats force them to write special-case code. Test the contract.**

```python
def test_all_error_responses_have_detail_key(client: TestClient):
    """Every 4xx response must contain a 'detail' field."""
    scenarios = [
        ("get",    "/tasks/99999",                 None),          # 404
        ("post",   "/tasks",                        None),          # 422 (no body)
        ("get",    "/tasks/abc",                    None),          # 422 (bad type)
        ("delete", "/tasks/99999",                  None),          # 404
    ]
    for method, url, body in scenarios:
        fn = getattr(client, method)
        response = fn(url) if body is None else fn(url, json=body)

        assert response.status_code >= 400, f"Expected error for {method.upper()} {url}"
        body_data = response.json()
        assert "detail" in body_data, (
            f"{method.upper()} {url} → {response.status_code} "
            f"but response missing 'detail' key. Got: {body_data}"
        )


def test_404_detail_is_string(client: TestClient):
    """HTTPException detail should be a plain string — not a nested object."""
    response = client.get("/tasks/99999")
    assert response.status_code == 404
    assert isinstance(response.json()["detail"], str)


def test_422_detail_is_list(client: TestClient):
    """Pydantic 422 detail is always a list of error objects."""
    response = client.post("/tasks", json={"title": ""})
    assert response.status_code == 422
    assert isinstance(response.json()["detail"], list)
    assert len(response.json()["detail"]) >= 1
```

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY ERROR CONSISTENCY MATTERS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FastAPI's DEFAULT error format:                                │
│  ├─ HTTPException (404, 403...)  → {"detail": "string"}         │
│  └─ Validation (422)            → {"detail": [{loc, msg, ...}]} │
│                                                                 │
│  If you use a custom exception handler that returns a           │
│  DIFFERENT shape — e.g., {"error": "not_found"} — your         │
│  frontend now has to check both formats. That's a bug.          │
│                                                                 │
│  The test above catches this automatically. If you add a        │
│  custom handler that breaks the shape, the test fails.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your app has POST /tasks (create) and DELETE /tasks/{id}.      │
│  Write a test that creates a task, deletes it, then verifies    │
│  that a GET on the deleted task returns 404 — and that a        │
│  second DELETE on the same ID also returns 404, not 204.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION

def test_deleted_task_is_truly_gone(client: TestClient):
    # Arrange
    create_resp = client.post("/tasks", json={"title": "Temporary"})
    assert create_resp.status_code == 201
    task_id = create_resp.json()["id"]

    # Act — delete it
    delete_resp = client.delete(f"/tasks/{task_id}")
    assert delete_resp.status_code == 204

    # Assert — GET confirms it's gone
    get_resp = client.get(f"/tasks/{task_id}")
    assert get_resp.status_code == 404

    # Assert — second DELETE is consistent (idempotent behavior)
    second_delete = client.delete(f"/tasks/{task_id}")
    assert second_delete.status_code == 404
```

---

## 3.8 Rate Limiting, Pagination Preview, and OpenAPI Schema

### Testing Rate Limiting (429 Responses)

**Rate limiters return 429 Too Many Requests when a client sends too many requests in a window. You need to test that your limiter fires correctly — and that it includes the `Retry-After` header.**

```python
# This pattern assumes slowapi or a custom rate limiter is configured.
# The override strategy: override the rate limit key function or
# set the limit very low in the test configuration.

def test_rate_limited_request_returns_429(client: TestClient):
    """Exceeding the rate limit should return 429, not 200."""
    # Set limit to 3/minute in test config — send 4 requests
    for i in range(3):
        response = client.get("/tasks")
        assert response.status_code == 200

    # 4th request should be rejected
    response = client.get("/tasks")
    assert response.status_code == 429


def test_rate_limit_response_has_retry_after(client: TestClient):
    """429 response should tell clients when to retry."""
    for _ in range(3):
        client.get("/tasks")

    response = client.get("/tasks")
    assert response.status_code == 429
    # Retry-After tells the client how many seconds to wait
    assert "retry-after" in response.headers
    assert int(response.headers["retry-after"]) > 0
```

> "Rate limiting testing is integration-heavy — you need the real limiter configured. In unit tests, you can override the rate limiter dependency to return a fake that always allows or always rejects. The important thing is to TEST BOTH paths: allowed and rejected."

---

### Pagination Preview

**Pagination is covered in depth in Week 4, Lecture 2. But you need to know NOW what to test so you can write tests alongside your implementation.**

```python
# Assume your endpoint returns paginated data in this shape:
# {
#   "items": [...],
#   "total": 100,
#   "page": 1,
#   "size": 20,
#   "pages": 5
# }

def test_paginated_response_has_required_keys(client: TestClient):
    """Pagination metadata must be present on every paginated response."""
    response = client.get("/tasks", params={"page": 1, "size": 10})
    assert response.status_code == 200
    data = response.json()

    required_keys = {"items", "total", "page", "size"}
    assert required_keys.issubset(data.keys()), (
        f"Missing pagination keys: {required_keys - data.keys()}"
    )


def test_pagination_respects_size_param(client: TestClient):
    """size param should limit the number of items returned."""
    for i in range(5):
        client.post("/tasks", json={"title": f"Task {i}"})

    response = client.get("/tasks", params={"page": 1, "size": 2})
    assert len(response.json()["items"]) <= 2


def test_page_beyond_last_returns_empty_items(client: TestClient):
    """Requesting a page past the end should return items=[] not 404."""
    response = client.get("/tasks", params={"page": 9999, "size": 10})
    assert response.status_code == 200
    assert response.json()["items"] == []
```

---

### Testing OpenAPI Schema Generation

**FastAPI auto-generates your OpenAPI schema from your route definitions and Pydantic models. Test it — it's your API's public contract, and it can silently drift from your implementation.**

```python
def test_openapi_schema_endpoint_returns_200(client: TestClient):
    """The schema endpoint must always be accessible."""
    response = client.get("/openapi.json")
    assert response.status_code == 200


def test_openapi_schema_has_required_structure(client: TestClient):
    """Schema must follow OpenAPI 3.x structure."""
    response = client.get("/openapi.json")
    schema = response.json()

    assert "openapi" in schema
    assert schema["openapi"].startswith("3.")
    assert "info" in schema
    assert "paths" in schema


def test_openapi_documents_all_endpoints(client: TestClient):
    """Every endpoint must appear in the OpenAPI schema."""
    schema = client.get("/openapi.json").json()
    paths = schema["paths"]

    assert "/tasks" in paths
    assert "/tasks/{task_id}" in paths

    # Check HTTP methods are documented
    assert "post" in paths["/tasks"]    # Create
    assert "get" in paths["/tasks"]     # List


def test_response_model_is_reflected_in_schema(client: TestClient):
    """TaskResponse fields must appear in the schema components."""
    schema = client.get("/openapi.json").json()
    schemas = schema.get("components", {}).get("schemas", {})

    # TaskResponse should be in the schemas
    assert "TaskResponse" in schemas
    properties = schemas["TaskResponse"]["properties"]

    expected_fields = {"id", "title", "priority", "completed"}
    assert expected_fields.issubset(properties.keys())
```

> "If you rename a field in `TaskResponse` from `priority` to `importance`, the last test above fails immediately. You know before your frontend team does. That's the contract being enforced."

---

# PART 4: ISOLATING YOUR CODE

## 4.1 The Shared State Problem

**Let's run multiple tests and see what happens.**

```python
# test_tasks.py — ❌ These tests INTERFERE with each other
from fastapi.testclient import TestClient
from app import app

client = TestClient(app)


def test_first_task_gets_id_1():
    response = client.post("/tasks", json={"title": "First"})
    assert response.json()["id"] == 1  # ✅ Passes


def test_second_task_also_gets_id_1():
    response = client.post("/tasks", json={"title": "Second"})
    assert response.json()["id"] == 1  # ❌ FAILS! id is 2!
```

**Run it:**

```
$ pytest test_tasks.py -v

test_first_task_gets_id_1 PASSED
test_second_task_also_gets_id_1 FAILED
    assert response.json()["id"] == 1
    AssertionError: assert 2 == 1
```

**What happened?**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE SHARED STATE PROBLEM                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  test_first_task:                                               │
│  ── tasks_db is empty, _id_counter is 0                         │
│  ── Creates task → _id_counter becomes 1                        │
│  ── tasks_db now has {1: {...}}                                  │
│  ── ✅ id == 1                                                   │
│                                                                 │
│  test_second_task:                                              │
│  ── tasks_db STILL has {1: {...}} from previous test!           │
│  ── _id_counter is STILL 1                                      │
│  ── Creates task → _id_counter becomes 2                        │
│  ── tasks_db now has {1: {...}, 2: {...}}                        │
│  ── ❌ id == 2, not 1                                            │
│                                                                 │
│  The module-level tasks_db and _id_counter PERSIST between      │
│  tests. State from test 1 LEAKED into test 2.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "Even worse: what happens if you run these tests in a different ORDER? `test_second` runs first? Now `test_first` passes and `test_second` fails. Your tests are ORDER-DEPENDENT. This is a testing nightmare."

**The quick hack (don't do this):**

```python
# ❌ Fragile: reaches into app internals to reset state
from app import tasks_db, _id_counter

@pytest.fixture(autouse=True)
def reset_db():
    tasks_db.clear()
    # Can't even reset _id_counter — it's a module-level int!
    yield
    tasks_db.clear()
```

This is fragile because you need to know every piece of internal state. If the app adds another module-level variable, your reset breaks silently.

**The real solution: dependency injection.**

---

## 4.2 dependency_overrides (FastAPI's Secret Weapon)

**Step 1: Refactor the app to use a dependency for storage.**

> "Connection to Week 3 Lecture 4: you learned `Depends()`. Now you'll see WHY dependency injection matters — not just for code organization, but for TESTABILITY."

```python
# storage.py — Extract storage into its own module
class TaskStorage:
    def __init__(self) -> None:
        self._tasks: dict[int, dict] = {}
        self._next_id: int = 0
    
    def create(self, data: dict) -> dict:
        self._next_id += 1
        data["id"] = self._next_id
        self._tasks[self._next_id] = data
        return data
    
    def get(self, task_id: int) -> dict | None:
        return self._tasks.get(task_id)
    
    def list_all(self) -> list[dict]:
        return list(self._tasks.values())
```

```python
# app.py — Refactored with dependency injection
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, Field
from storage import TaskStorage

app = FastAPI()

# Module-level instance (production)
_storage = TaskStorage()


def get_storage() -> TaskStorage:
    """Dependency: provides the storage instance."""
    return _storage


class TaskCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    priority: int = Field(ge=1, le=5, default=3)


class TaskResponse(BaseModel):
    id: int
    title: str
    priority: int
    completed: bool


@app.post("/tasks", status_code=201)
async def create_task(
    task: TaskCreate,
    storage: TaskStorage = Depends(get_storage),
) -> TaskResponse:
    task_data = {**task.model_dump(), "completed": False}
    created = storage.create(task_data)
    return TaskResponse(**created)


@app.get("/tasks/{task_id}")
async def get_task(
    task_id: int,
    storage: TaskStorage = Depends(get_storage),
) -> TaskResponse:
    task_data = storage.get(task_id)
    if task_data is None:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")
    return TaskResponse(**task_data)
```

**Step 2: Override the dependency in tests.**

```python
# test_tasks.py — ✅ Isolated tests with dependency_overrides
import pytest
from fastapi.testclient import TestClient
from app import app, get_storage
from storage import TaskStorage


@pytest.fixture
def client():
    """Each test gets a FRESH storage instance."""
    test_storage = TaskStorage()   # Brand new, empty storage
    
    # Tell FastAPI: when anything asks for get_storage, 
    # give them test_storage instead
    app.dependency_overrides[get_storage] = lambda: test_storage
    
    yield TestClient(app)
    
    # Clean up: remove the override
    app.dependency_overrides.clear()


def test_first_task_gets_id_1(client: TestClient):
    response = client.post("/tasks", json={"title": "First"})
    assert response.json()["id"] == 1  # ✅ Fresh storage!


def test_second_task_also_gets_id_1(client: TestClient):
    response = client.post("/tasks", json={"title": "Second"})
    assert response.json()["id"] == 1  # ✅ ALSO 1! Fresh storage each time!
```

**Both pass. Every test starts clean.**

```
$ pytest test_tasks.py -v

test_first_task_gets_id_1 PASSED
test_second_task_also_gets_id_1 PASSED
```

**How `dependency_overrides` works:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 dependency_overrides                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  app.dependency_overrides is a DICT:                            │
│  {                                                              │
│    original_function: replacement_function                      │
│  }                                                              │
│                                                                 │
│  When FastAPI resolves Depends(get_storage):                    │
│                                                                 │
│  PRODUCTION (no overrides):                                     │
│  Depends(get_storage)                                           │
│       │                                                         │
│       ▼                                                         │
│  get_storage()  →  returns _storage (module-level, shared)      │
│                                                                 │
│                                                                 │
│  TEST (with override):                                          │
│  app.dependency_overrides[get_storage] = lambda: test_storage   │
│                                                                 │
│  Depends(get_storage)                                           │
│       │                                                         │
│       ▼                                                         │
│  Override found! Call lambda instead                             │
│       │                                                         │
│       ▼                                                         │
│  lambda: test_storage  →  returns test_storage (fresh, per-test)│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In the vending machine analogy:**

> "`dependency_overrides` is like opening the machine's service panel and swapping the product tray. Instead of whatever random stock was left from previous customers, you load a KNOWN set of test products into KNOWN slots. Every test starts from the exact same state. That's isolation."

---

## 4.3 Overriding Yield Dependencies

**Connection to Week 3 Lecture 4:**

> "You learned yield dependencies for setup/teardown patterns. Overriding them works exactly the same way — the override can also be a generator."

```python
# In your app — a yield dependency
def get_audit_logger():
    """Provides an audit logger; flushes on teardown."""
    logger = AuditLogger(destination="production.log")
    try:
        yield logger
    finally:
        logger.flush()

@app.post("/tasks", status_code=201)
async def create_task(
    task: TaskCreate,
    storage: TaskStorage = Depends(get_storage),
    audit: AuditLogger = Depends(get_audit_logger),
) -> TaskResponse:
    created = storage.create({**task.model_dump(), "completed": False})
    audit.log(f"Task created: {created['id']}")
    return TaskResponse(**created)
```

```python
# In your tests — override with a test version
def override_audit_logger():
    """Test version: logs to memory, not to file."""
    logger = FakeAuditLogger()  # Stores logs in a list
    try:
        yield logger
    finally:
        pass  # No flush needed in tests


@pytest.fixture
def client():
    test_storage = TaskStorage()
    app.dependency_overrides[get_storage] = lambda: test_storage
    app.dependency_overrides[get_audit_logger] = override_audit_logger
    
    yield TestClient(app)
    
    app.dependency_overrides.clear()
```

**The override must match the pattern:**

```
┌─────────────────────────────────────────────────────────────────┐
│            MATCHING DEPENDENCY PATTERNS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Original uses return → Override must use return (or lambda)    │
│                                                                 │
│      def get_storage():                                         │
│          return storage                                         │
│                                                                 │
│      override = lambda: test_storage     ✅                     │
│                                                                 │
│                                                                 │
│  Original uses yield → Override should use yield                │
│                                                                 │
│      def get_logger():                                          │
│          logger = Logger()                                      │
│          yield logger                                           │
│          logger.close()                                         │
│                                                                 │
│      def override_logger():                                     │
│          yield FakeLogger()              ✅                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "You'll use yield dependency overrides constantly in Week 6 when you inject database sessions. The pattern: production yields a real session that commits; tests yield a session that rolls back. Same interface, different behavior."

---

## 4.4 Unit vs Integration: When to Override, When Not To

**This is a judgment call, not a rule. But here's the framework.**

```
┌─────────────────────────────────────────────────────────────────┐
│             UNIT vs INTEGRATION TESTS FOR APIs                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  UNIT TEST (override dependencies)                              │
│  ─────────────────────────────────                              │
│  Replace real dependencies with fakes/mocks.                    │
│  Test your endpoint logic IN ISOLATION.                         │
│                                                                 │
│  ├─ Fast (no real I/O)                                          │
│  ├─ Deterministic (no flaky network/database)                   │
│  ├─ Pinpoints failures (if it breaks, it's YOUR code)           │
│  └─ Risk: your fakes might not match real behavior              │
│                                                                 │
│  Example: Override get_storage with an in-memory fake           │
│                                                                 │
│                                                                 │
│  INTEGRATION TEST (use real dependencies)                       │
│  ────────────────────────────────────────                       │
│  Use the actual dependencies (real database, real service).     │
│  Test that everything works TOGETHER.                           │
│                                                                 │
│  ├─ Realistic (catches integration bugs)                        │
│  ├─ Slower (real I/O, setup/teardown)                           │
│  ├─ Can be flaky (network, timing)                              │
│  └─ Harder to pinpoint failures (which layer broke?)            │
│                                                                 │
│  Example: Use a real test database (Week 6)                     │
│                                                                 │
│                                                                 │
│  THE TESTING PYRAMID (from Week 2):                             │
│                                                                 │
│          /\                                                     │
│         /  \        Few end-to-end tests (slow, realistic)      │
│        /    \                                                   │
│       /──────\      Some integration tests (medium)             │
│      /        \                                                 │
│     /──────────\    Many unit tests (fast, isolated)            │
│    /            \                                               │
│                                                                 │
│  For your Week 3-4 project (in-memory storage):                 │
│  Everything is effectively a unit test — there's no external    │
│  system to integrate with yet. That changes in Week 6.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 Overrides vs Mocks (Connection to Week 2)

**Connection to what you've learned:**

> "In Week 2, you learned `unittest.mock` for creating test doubles. How does that compare to `dependency_overrides`? They solve the same problem — isolation — but at different levels."

```
┌─────────────────────────────────────────────────────────────────┐
│          dependency_overrides vs unittest.mock                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  dependency_overrides                                           │
│  ────────────────────                                           │
│  FastAPI-specific. Replaces an entire dependency function.      │
│  Works at the FRAMEWORK level — FastAPI knows about it.         │
│  Clean, explicit, purpose-built for API testing.                │
│                                                                 │
│    app.dependency_overrides[get_storage] = lambda: fake         │
│                                                                 │
│  Best for: replacing dependencies injected via Depends()        │
│                                                                 │
│                                                                 │
│  unittest.mock (Mock, patch)                                    │
│  ───────────────────────────                                    │
│  General-purpose. Replaces any object/function at runtime.      │
│  Works at the PYTHON level — patches module attributes.         │
│  Flexible but can lead to brittle tests.                        │
│                                                                 │
│    with patch("app.some_service") as mock_service:              │
│        mock_service.return_value = fake_result                  │
│                                                                 │
│  Best for: replacing things NOT injected via Depends()          │
│  (external libraries, module-level functions, etc.)             │
│                                                                 │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  If it's a Depends() → use dependency_overrides                 │
│  If it's NOT a Depends() → use unittest.mock                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Example: combining both in one test:**

```python
from unittest.mock import patch

def test_create_task_with_notification(client: TestClient):
    """Test that creating a task sends a notification."""
    
    # dependency_overrides already handle storage (via fixture)
    
    # unittest.mock handles the notification service
    # (not injected via Depends — it's a module-level call)
    with patch("app.send_notification") as mock_notify:
        response = client.post("/tasks", json={"title": "Important"})
        
        assert response.status_code == 201
        mock_notify.assert_called_once()
```

---

## 4.6 Testing Authenticated Endpoints

**Connection to Week 3 Lecture 4:**

> "You learned `Depends()` for injecting dependencies. `get_current_user` is just a dependency — which means `dependency_overrides` is the perfect tool for auth testing. No real tokens. No JWT decode. No database lookup. You inject a known user and test what your endpoint does WITH that user."

**The app code with auth (Week 9 will cover the full implementation):**

```python
# app.py extension — simplified auth for testing demonstration
from pydantic import BaseModel
from fastapi.security import OAuth2PasswordBearer
from fastapi import Depends, HTTPException, status

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")


class CurrentUser(BaseModel):
    id: int
    username: str
    role: str   # "user" | "admin"


async def get_current_user(
    token: str = Depends(oauth2_scheme)
) -> CurrentUser:
    """Decode token and return current user. Full implementation in Week 9."""
    # ... JWT decode, DB lookup ...
    raise NotImplementedError  # Placeholder


async def require_admin(
    current_user: CurrentUser = Depends(get_current_user),
) -> CurrentUser:
    """Secondary dependency: requires admin role."""
    if current_user.role != "admin":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin access required"
        )
    return current_user


@app.get("/me")
async def get_me(current_user: CurrentUser = Depends(get_current_user)):
    return {"id": current_user.id, "username": current_user.username}


@app.delete("/admin/users/{user_id}", status_code=204)
async def delete_user(
    user_id: int,
    _: CurrentUser = Depends(require_admin),   # admin only
):
    ...  # delete logic
```

**Now the test file:**

```python
# tests/test_auth.py
import pytest
from fastapi.testclient import TestClient
from app import app, get_current_user, CurrentUser


# ── Helper ──────────────────────────────────────────────────────
def fake_user(role: str = "user", username: str = "testuser") -> CurrentUser:
    return CurrentUser(id=42, username=username, role=role)


# ── Fixtures ─────────────────────────────────────────────────────
@pytest.fixture
def unauth_client():
    """No auth override — requests arrive without a valid token."""
    # NOTE: do NOT override get_current_user here.
    # OAuth2PasswordBearer raises 401 if the Authorization header is absent.
    yield TestClient(app)


@pytest.fixture
def user_client():
    """Authenticated as a regular user (role='user')."""
    app.dependency_overrides[get_current_user] = lambda: fake_user(role="user")
    yield TestClient(app)
    app.dependency_overrides.clear()


@pytest.fixture
def admin_client():
    """Authenticated as an admin user (role='admin')."""
    app.dependency_overrides[get_current_user] = lambda: fake_user(role="admin")
    yield TestClient(app)
    app.dependency_overrides.clear()


# ── Tests ─────────────────────────────────────────────────────────
def test_no_token_returns_401(unauth_client: TestClient):
    """Unauthenticated request to a protected endpoint → 401."""
    response = unauth_client.get("/me")
    assert response.status_code == 401


def test_authenticated_user_sees_own_profile(user_client: TestClient):
    """Valid user → 200 with correct username."""
    response = user_client.get("/me")
    assert response.status_code == 200
    assert response.json()["username"] == "testuser"


def test_regular_user_cannot_access_admin_endpoint(user_client: TestClient):
    """User with role='user' → 403 Forbidden on admin route."""
    response = user_client.delete("/admin/users/99")
    assert response.status_code == 403
    assert "Admin" in response.json()["detail"]


def test_admin_can_access_admin_endpoint(admin_client: TestClient):
    """User with role='admin' → allowed on admin route."""
    response = admin_client.delete("/admin/users/99")
    assert response.status_code == 204
```

**Testing user-specific data isolation:**

```python
def test_user_can_only_see_own_tasks(client: TestClient, storage: TaskStorage):
    """User A should not see User B's tasks."""
    # Simulate User A creating a task
    app.dependency_overrides[get_current_user] = lambda: fake_user(id=1, username="alice")
    with TestClient(app) as alice_client:
        alice_client.post("/tasks", json={"title": "Alice's secret task"})

    # Simulate User B listing tasks
    app.dependency_overrides[get_current_user] = lambda: fake_user(id=2, username="bob")
    with TestClient(app) as bob_client:
        response = bob_client.get("/tasks")
        task_titles = [t["title"] for t in response.json()]
        assert "Alice's secret task" not in task_titles

    app.dependency_overrides.clear()
```

```
┌─────────────────────────────────────────────────────────────────┐
│           AUTH TESTING — WHAT YOU'RE ACTUALLY TESTING           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NOT testing: JWT signature validity                            │
│  NOT testing: Password hashing correctness                      │
│  NOT testing: Token expiry                                      │
│  (Those are unit tests on your auth utility functions)          │
│                                                                 │
│  YOU ARE testing:                                               │
│  ✅ That unauthenticated requests are rejected (401)            │
│  ✅ That unauthorized roles are rejected (403)                  │
│  ✅ That authorized roles can proceed (200/204)                 │
│  ✅ That user-specific data isolation works correctly           │
│  ✅ That the dependency chain (get_current_user →               │
│       require_admin) resolves correctly                         │
│                                                                 │
│  This is the HTTP contract for authentication.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You have an endpoint PATCH /tasks/{id} that requires the       │
│  current user to be the OWNER of the task (user.id must match   │
│  task.owner_id), otherwise return 403. Write TWO fixtures and   │
│  TWO tests: one showing the owner can update, and one showing   │
│  a different user cannot.                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION

@pytest.fixture
def owner_client():
    app.dependency_overrides[get_current_user] = lambda: fake_user(id=1)
    yield TestClient(app)
    app.dependency_overrides.clear()


@pytest.fixture
def other_client():
    app.dependency_overrides[get_current_user] = lambda: fake_user(id=2)
    yield TestClient(app)
    app.dependency_overrides.clear()


def test_owner_can_update_task(owner_client, storage):
    # Create a task owned by user id=1
    task = storage.create({"title": "My task", "priority": 3,
                           "completed": False, "owner_id": 1})
    response = owner_client.patch(
        f"/tasks/{task['id']}",
        json={"title": "Updated title"}
    )
    assert response.status_code == 200


def test_non_owner_cannot_update_task(other_client, storage):
    task = storage.create({"title": "Not yours", "priority": 3,
                           "completed": False, "owner_id": 1})
    response = other_client.patch(
        f"/tasks/{task['id']}",
        json={"title": "Hacked title"}
    )
    assert response.status_code == 403
```

---

## 4.7 Testing Middleware

**The pipeline diagram from Part 1 shows middleware as the outermost layer. You cannot test middleware by calling functions directly — you MUST go through the HTTP pipeline. TestClient is the only way.**

**App code (middleware extension):**

```python
# app.py extension
import time
from starlette.middleware.base import BaseHTTPMiddleware
from fastapi import Request

class RequestIDMiddleware(BaseHTTPMiddleware):
    """Echoes X-Request-ID from request to response; generates one if absent."""
    async def dispatch(self, request: Request, call_next):
        request_id = request.headers.get("X-Request-ID", "auto-generated")
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response


class TimingMiddleware(BaseHTTPMiddleware):
    """Adds X-Process-Time (ms) to every response."""
    async def dispatch(self, request: Request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        elapsed_ms = (time.perf_counter() - start) * 1000
        response.headers["X-Process-Time"] = f"{elapsed_ms:.2f}ms"
        return response


app.add_middleware(TimingMiddleware)
app.add_middleware(RequestIDMiddleware)
```

**Tests:**

```python
# tests/test_middleware.py

def test_request_id_echoed_when_provided(client: TestClient):
    """Provided X-Request-ID should appear verbatim in response headers."""
    response = client.get("/tasks", headers={"X-Request-ID": "trace-abc-123"})
    assert response.headers.get("x-request-id") == "trace-abc-123"


def test_request_id_generated_when_absent(client: TestClient):
    """Missing X-Request-ID → middleware adds a generated value."""
    response = client.get("/tasks")
    assert "x-request-id" in response.headers
    assert response.headers["x-request-id"] == "auto-generated"


def test_timing_header_present_on_all_responses(client: TestClient):
    """Every response — success and error — should have X-Process-Time."""
    for path in ["/tasks", "/tasks/99999", "/tasks/abc"]:
        response = client.get(path)
        assert "x-process-time" in response.headers, (
            f"X-Process-Time missing on GET {path} → {response.status_code}"
        )


def test_timing_header_is_valid_number(client: TestClient):
    """X-Process-Time must parse as a non-negative float."""
    response = client.get("/tasks")
    raw = response.headers["x-process-time"]
    # Strip unit suffix if present ("12.34ms" → 12.34)
    value = float(raw.rstrip("ms"))
    assert value >= 0


def test_middleware_runs_on_error_responses(client: TestClient):
    """Middleware must wrap error responses too — not just successes."""
    response = client.get("/tasks/99999")
    assert response.status_code == 404
    # Middleware should have run even on this error response
    assert "x-request-id" in response.headers
    assert "x-process-time" in response.headers
```

> "The last test is the most important. A common middleware bug: the middleware calls `call_next()` but only modifies the response on the HAPPY path. If an exception short-circuits `call_next`, the middleware's response modification never runs. TestClient lets you catch this."

---

## 4.8 Mocking External HTTP Calls

**Real APIs call other APIs. You cannot let those calls happen during tests — the network is slow, flaky, costs money, and makes tests non-deterministic. You must mock the external HTTP layer.**

**Two tools: `respx` (httpx-specific, purpose-built) and `unittest.mock` (general-purpose).**

**App code (external API integration):**

```python
# app.py extension
import httpx

WEATHER_API_BASE = "https://api.openweathermap.org/data/2.5/weather"
WEATHER_API_KEY = "your-api-key"

@app.get("/weather/{city}")
async def get_weather(city: str) -> dict:
    """Fetch weather from OpenWeatherMap and return simplified data."""
    async with httpx.AsyncClient() as http:
        resp = await http.get(
            WEATHER_API_BASE,
            params={"q": city, "appid": WEATHER_API_KEY, "units": "metric"}
        )
    if resp.status_code == 404:
        raise HTTPException(
            status_code=404,
            detail=f"City '{city}' not found"
        )
    if resp.status_code != 200:
        raise HTTPException(
            status_code=502,
            detail="Weather service unavailable"
        )
    data = resp.json()
    return {
        "city": city,
        "temperature": data["main"]["temp"],
        "description": data["weather"][0]["description"],
    }
```

---

### Using respx

**Setup:**

```toml
# pyproject.toml
[project.optional-dependencies]
test = ["pytest", "pytest-asyncio", "httpx", "respx"]
```

**Tests:**

```python
# tests/test_weather.py
import respx
import httpx

WEATHER_API_BASE = "https://api.openweathermap.org/data/2.5/weather"

FAKE_WEATHER_RESPONSE = {
    "main": {"temp": 18.5},
    "weather": [{"description": "partly cloudy"}]
}


@respx.mock
def test_get_weather_success(client: TestClient):
    """Happy path: external API responds → our endpoint returns shaped data."""
    respx.get(WEATHER_API_BASE).mock(
        return_value=httpx.Response(200, json=FAKE_WEATHER_RESPONSE)
    )

    response = client.get("/weather/London")

    assert response.status_code == 200
    data = response.json()
    assert data["city"] == "London"
    assert data["temperature"] == 18.5
    assert data["description"] == "partly cloudy"


@respx.mock
def test_unknown_city_returns_404(client: TestClient):
    """When external API returns 404 for city, our endpoint should too."""
    respx.get(WEATHER_API_BASE).mock(
        return_value=httpx.Response(404, json={"message": "city not found"})
    )

    response = client.get("/weather/MadeUpCity")

    assert response.status_code == 404
    assert "MadeUpCity" in response.json()["detail"]


@respx.mock
def test_weather_service_down_returns_502(client: TestClient):
    """When external API fails (500), our endpoint should return 502."""
    respx.get(WEATHER_API_BASE).mock(
        return_value=httpx.Response(500, json={"message": "internal error"})
    )

    response = client.get("/weather/London")

    assert response.status_code == 502
    assert "unavailable" in response.json()["detail"].lower()


@respx.mock
def test_correct_api_params_are_sent(client: TestClient):
    """Verify our endpoint sends the correct params to the external API."""
    mock_weather = respx.get(WEATHER_API_BASE).mock(
        return_value=httpx.Response(200, json=FAKE_WEATHER_RESPONSE)
    )

    client.get("/weather/Paris")

    assert mock_weather.called
    request_url = str(mock_weather.calls.last.request.url)
    assert "Paris" in request_url
    assert "metric" in request_url    # units=metric
```

---

### Using unittest.mock as an alternative

```python
# When respx is not available or you need to mock at the object level
from unittest.mock import patch, AsyncMock, MagicMock

def test_weather_with_unittest_mock(client: TestClient):
    """Same test using unittest.mock instead of respx."""
    mock_response = MagicMock()
    mock_response.status_code = 200
    mock_response.json.return_value = FAKE_WEATHER_RESPONSE

    # Patch the AsyncClient context manager
    mock_http = AsyncMock()
    mock_http.get = AsyncMock(return_value=mock_response)

    with patch("app.httpx.AsyncClient") as mock_client_cls:
        mock_client_cls.return_value.__aenter__ = AsyncMock(return_value=mock_http)
        mock_client_cls.return_value.__aexit__ = AsyncMock(return_value=False)

        response = client.get("/weather/London")

    assert response.status_code == 200
    assert response.json()["temperature"] == 18.5
```

```
┌─────────────────────────────────────────────────────────────────┐
│              respx vs unittest.mock                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  respx                                                          │
│  ── Intercepts at the httpx transport level                     │
│  ── URL/method matching is first-class                          │
│  ── Inspect actual requests (url, headers, body)                │
│  ── Clean decorator or context manager API                      │
│  ── ✅ Use when your app uses httpx                             │
│                                                                 │
│  unittest.mock                                                  │
│  ── Patches Python objects at module level                      │
│  ── Verbose for async context managers (nested AsyncMock)       │
│  ── Works with any HTTP library (requests, aiohttp, httpx)      │
│  ── ✅ Use when respx isn't available or you mock non-httpx code │
│                                                                 │
│  RULE: Your app uses httpx → use respx. It's purpose-built.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your endpoint GET /exchange-rate/{currency} calls              │
│  https://api.exchangerate.host/latest?base={currency}.          │
│  Using respx, write a test that: (1) mocks a successful         │
│  response returning a rate of 1.25, and (2) verifies the        │
│  mock was called with the correct currency in the URL.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION
import respx
import httpx

EXCHANGE_API = "https://api.exchangerate.host/latest"

@respx.mock
def test_exchange_rate_success(client: TestClient):
    mock_api = respx.get(EXCHANGE_API).mock(
        return_value=httpx.Response(
            200,
            json={"base": "USD", "rates": {"EUR": 1.25}}
        )
    )

    response = client.get("/exchange-rate/USD")

    assert response.status_code == 200
    assert response.json()["rate"] == 1.25

    # Verify correct params were sent
    assert mock_api.called
    request_url = str(mock_api.calls.last.request.url)
    assert "USD" in request_url
```

---

# PART 5: ASYNC TESTING & TEST ARCHITECTURE

## 5.1 When TestClient Isn't Enough

**TestClient handles 90% of API testing. But it's synchronous under the hood.**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHEN TO MOVE BEYOND TestClient                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TestClient is FINE when:                                       │
│  ├─ Testing request/response cycles (GET, POST, etc.)           │
│  ├─ Testing validation, status codes, response bodies           │
│  ├─ Testing dependency overrides                                │
│  └─ ✅ This covers MOST of your Week 3-4 project tests          │
│                                                                 │
│  You NEED async testing when:                                   │
│  ├─ Testing WebSocket endpoints (Week 12)                       │
│  ├─ Testing Server-Sent Events (SSE)                            │
│  ├─ Your fixtures are async (async database sessions — Week 6)  │
│  ├─ You want to test concurrent request behavior                │
│  └─ Your test itself needs to do async I/O                      │
│                                                                 │
│  DON'T use async tests just because your endpoints are          │
│  async def. TestClient handles that transparently.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 httpx.AsyncClient with ASGITransport

**When you do need async tests, `httpx.AsyncClient` is the tool.**

**Setup in `pyproject.toml`:**

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"

[project.optional-dependencies]
test = [
    "pytest",
    "pytest-asyncio",
    "httpx",
]
```

### pytest-asyncio Modes

**Paying off tech debt from Week 2, Lecture 2: `asyncio_mode = "auto"` has been in your pyproject.toml. Here's exactly what it controls.**

```
┌─────────────────────────────────────────────────────────────────┐
│              pytest-asyncio MODES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MODE: "auto"                                                   │
│  ── Any async def test_* runs as an asyncio test automatically  │
│  ── No extra decorator required                                 │
│  ── Lowest friction                                             │
│  ── ✅ Use this — it's the right default for async-heavy APIs   │
│                                                                 │
│  MODE: "strict"                                                 │
│  ── Only test functions decorated with @pytest.mark.asyncio     │
│     are treated as async tests                                  │
│  ── All others are treated as synchronous (even if async def)   │
│  ── Requires more ceremony but is explicit                      │
│  ── ✅ Use in large teams needing fine-grained control          │
│                                                                 │
│  MODE: "legacy"  (before pytest-asyncio 0.21)                   │
│  ── Old auto-detection heuristic                                │
│  ── Deprecated — do not use in new projects                     │
│  ── ⚠️ You will encounter this in old codebases                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In code, the difference:**

```python
# ── asyncio_mode = "auto" (your pyproject.toml is already set this way) ──
# No decorator needed. Write async def and it just works.

async def test_create_task(async_client: AsyncClient):
    response = await async_client.post("/tasks", json={"title": "Auto mode"})
    assert response.status_code == 201


# ── asyncio_mode = "strict" (if you switch to strict later) ──
# Must explicitly mark EVERY async test function.

@pytest.mark.asyncio
async def test_create_task(async_client: AsyncClient):
    response = await async_client.post("/tasks", json={"title": "Strict mode"})
    assert response.status_code == 201

# In strict mode, an unmarked async def test is collected but
# pytest detects it never actually awaited — and raises a warning.
```

---

**Your first async API test:**

```python
# test_tasks_async.py
import pytest
from httpx import AsyncClient, ASGITransport
from app import app


@pytest.fixture
async def async_client():
    """Async test client — truly async, no sync wrapper."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client


async def test_create_task(async_client: AsyncClient):
    response = await async_client.post(
        "/tasks",
        json={"title": "Async test", "priority": 2}
    )
    
    assert response.status_code == 201
    assert response.json()["title"] == "Async test"
```

**Compare the two approaches:**

```
┌─────────────────────────────────────────────────────────────────┐
│            TestClient vs AsyncClient                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TestClient (sync):                                             │
│  ──────────────────                                             │
│     def test_create(client: TestClient):                        │
│         response = client.post("/tasks", json={...})            │
│         assert response.status_code == 201                      │
│                                                                 │
│  ├─ Plain def test function                                     │
│  ├─ No await needed                                             │
│  ├─ Simpler to write                                            │
│  └─ Uses requests-like interface                                │
│                                                                 │
│                                                                 │
│  AsyncClient (async):                                           │
│  ────────────────────                                           │
│     async def test_create(async_client: AsyncClient):           │
│         response = await async_client.post("/tasks", json={..}) │
│         assert response.status_code == 201                      │
│                                                                 │
│  ├─ async def test function                                     │
│  ├─ Must await every call                                       │
│  ├─ More ceremony, but async-native                             │
│  └─ Uses httpx interface (you'll learn httpx in Week 8)         │
│                                                                 │
│                                                                 │
│  RECOMMENDATION for your project:                               │
│  Start with TestClient. Switch to AsyncClient when              │
│  you have a concrete reason (async fixtures, WebSockets).       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Combining async client with dependency overrides:**

```python
import pytest
from httpx import AsyncClient, ASGITransport
from app import app, get_storage
from storage import TaskStorage


@pytest.fixture
async def async_client():
    test_storage = TaskStorage()
    app.dependency_overrides[get_storage] = lambda: test_storage
    
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client
    
    app.dependency_overrides.clear()


async def test_create_and_retrieve(async_client: AsyncClient):
    # Create
    create_resp = await async_client.post(
        "/tasks", json={"title": "Async roundtrip"}
    )
    task_id = create_resp.json()["id"]
    
    # Retrieve
    get_resp = await async_client.get(f"/tasks/{task_id}")
    assert get_resp.status_code == 200
    assert get_resp.json()["title"] == "Async roundtrip"
```

---

### Concurrent Request Testing

**TestClient is synchronous — you cannot send two requests "at the same time" with it. AsyncClient is truly async, which means `asyncio.gather()` works the way you'd expect.**

```python
# tests/test_concurrent.py
import asyncio
import pytest
from httpx import AsyncClient, ASGITransport
from app import app, get_storage
from storage import TaskStorage


@pytest.fixture
async def async_client():
    test_storage = TaskStorage()
    app.dependency_overrides[get_storage] = lambda: test_storage
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client
    app.dependency_overrides.clear()


async def test_concurrent_task_creation_assigns_unique_ids(
    async_client: AsyncClient,
):
    """10 concurrent POSTs must each get a unique ID — no race conditions."""
    coroutines = [
        async_client.post("/tasks", json={"title": f"Task {i}"})
        for i in range(10)
    ]
    responses = await asyncio.gather(*coroutines)

    # All requests succeeded
    for resp in responses:
        assert resp.status_code == 201, f"Unexpected status: {resp.status_code}"

    # All IDs are unique — no ID assigned twice
    ids = [r.json()["id"] for r in responses]
    assert len(set(ids)) == 10, f"Duplicate IDs detected: {ids}"


async def test_concurrent_reads_return_consistent_data(
    async_client: AsyncClient,
):
    """50 simultaneous reads of the same resource should all agree."""
    create_resp = await async_client.post(
        "/tasks", json={"title": "Shared Task"}
    )
    task_id = create_resp.json()["id"]

    reads = [async_client.get(f"/tasks/{task_id}") for _ in range(50)]
    responses = await asyncio.gather(*reads)

    for resp in responses:
        assert resp.status_code == 200
        assert resp.json()["id"] == task_id
        assert resp.json()["title"] == "Shared Task"
```

```
┌─────────────────────────────────────────────────────────────────┐
│              WHEN TO WRITE CONCURRENT TESTS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE concurrent tests when:                                     │
│  ├─ Testing ID generation or sequence counters                  │
│  ├─ Verifying mutable shared state under concurrent load        │
│  ├─ Checking that rate limiters fire at the correct count       │
│  └─ Testing that connection pool doesn't exhaust under load     │
│                                                                 │
│  DON'T use concurrent tests for:                                │
│  ├─ Validation errors — no concurrency needed                   │
│  ├─ Auth behavior — sequential tests are clearer                │
│  └─ Most standard CRUD operations                               │
│                                                                 │
│  IMPORTANT: Use AsyncClient for concurrent tests.               │
│  TestClient is synchronous — asyncio.gather() with TestClient   │
│  does not give you true concurrency.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Without looking above: write an async test that fires 5        │
│  concurrent DELETE requests to /tasks/1 and asserts that        │
│  exactly ONE returns 204 and the remaining FOUR return 404.     │
│  (This tests that your delete is truly idempotent under         │
│  concurrent load and doesn't double-delete or crash.)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION

async def test_concurrent_deletes_are_idempotent(async_client: AsyncClient):
    # Arrange: create one task
    create_resp = await async_client.post("/tasks", json={"title": "Delete race"})
    task_id = create_resp.json()["id"]

    # Act: 5 concurrent deletes
    deletes = [async_client.delete(f"/tasks/{task_id}") for _ in range(5)]
    responses = await asyncio.gather(*deletes)

    status_codes = [r.status_code for r in responses]

    # Assert: exactly one 204, four 404s
    assert status_codes.count(204) == 1, f"Expected 1×204, got: {status_codes}"
    assert status_codes.count(404) == 4, f"Expected 4×404, got: {status_codes}"
```

---

## 5.3 Test File Organization

**As your test suite grows, structure matters.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  TEST FILE STRUCTURE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  project/                                                       │
│  ├── src/                                                       │
│  │   ├── app.py                                                │
│  │   ├── storage.py                                            │
│  │   ├── models.py                                             │
│  │   └── ...                                                   │
│  │                                                              │
│  └── tests/                                                    │
│      ├── conftest.py          ← Shared fixtures (client, etc.) │
│      ├── factories.py         ← Test data generators           │
│      ├── test_tasks.py        ← Tests for /tasks endpoints     │
│      ├── test_categories.py   ← Tests for /categories          │
│      └── test_tags.py         ← Tests for /tags                │
│                                                                 │
│  RULE: one test file per RESOURCE (endpoint group).             │
│  RULE: shared fixtures go in conftest.py (Week 2 callback).    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**conftest.py — the shared fixture hub:**

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from app import app, get_storage
from storage import TaskStorage


@pytest.fixture
def storage() -> TaskStorage:
    """Fresh storage for each test."""
    return TaskStorage()


@pytest.fixture
def client(storage: TaskStorage) -> TestClient:
    """TestClient with isolated storage."""
    app.dependency_overrides[get_storage] = lambda: storage
    yield TestClient(app)
    app.dependency_overrides.clear()
```

**Now every test file gets `client` for free:**

```python
# tests/test_tasks.py — no fixture definitions needed!
from fastapi.testclient import TestClient


def test_create_task(client: TestClient):
    response = client.post("/tasks", json={"title": "Test"})
    assert response.status_code == 201


def test_list_tasks_empty(client: TestClient):
    response = client.get("/tasks")
    assert response.status_code == 200
    assert response.json() == []
```

> "Remember from Week 2: pytest automatically discovers `conftest.py` files and makes their fixtures available to all test files in the same directory and subdirectories. One fixture definition, used everywhere."

---

## 5.4 Factory Pattern for Test Data

**You'll write `json={"title": "Test", "priority": 3}` hundreds of times. Don't.**

```python
# tests/factories.py
from typing import Any


def make_task_payload(**overrides: Any) -> dict:
    """Build a valid task creation payload with sensible defaults.
    
    Override any field:
        make_task_payload(title="Custom", priority=5)
    """
    defaults = {
        "title": "Test Task",
        "priority": 3,
    }
    return {**defaults, **overrides}
```

**Usage in tests:**

```python
# tests/test_tasks.py
from factories import make_task_payload


def test_create_task(client):
    response = client.post("/tasks", json=make_task_payload())
    assert response.status_code == 201


def test_create_high_priority_task(client):
    payload = make_task_payload(title="Urgent bug", priority=5)
    response = client.post("/tasks", json=payload)
    assert response.json()["priority"] == 5


def test_create_task_with_long_title(client):
    payload = make_task_payload(title="a" * 100)  # Exactly at limit
    response = client.post("/tasks", json=payload)
    assert response.status_code == 201
```

**Why factories matter:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHY FACTORIES?                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT FACTORY:                                               │
│  ── Every test repeats the same boilerplate payload             │
│  ── Adding a required field to TaskCreate?                      │
│     Update 47 test files. Miss one? Test fails.                 │
│                                                                 │
│  WITH FACTORY:                                                  │
│  ── Tests only specify the fields THEY CARE ABOUT              │
│  ── Adding a required field? Update ONE factory.                │
│  ── Tests read clearly: "this test is about priority=5"         │
│     (the title doesn't matter, so the factory picks one)        │
│                                                                 │
│  Factories make tests:                                          │
│  ├─ DRY (Don't Repeat Yourself)                                 │
│  ├─ Resilient to model changes                                  │
│  └─ Focused on what matters (the thing being tested)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Helper fixture for pre-populated storage:**

```python
# tests/conftest.py — add a fixture that creates tasks for you

@pytest.fixture
def populated_client(client: TestClient) -> TestClient:
    """Client with 3 pre-existing tasks."""
    for i in range(1, 4):
        client.post("/tasks", json=make_task_payload(title=f"Task {i}"))
    return client
```

```python
# tests/test_tasks.py
def test_list_returns_all_tasks(populated_client):
    response = populated_client.get("/tasks")
    assert len(response.json()) == 3
```

---

## 5.5 Your Testing Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│             TESTING STRATEGY FOR YOUR PROJECT                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FOR EACH ENDPOINT, TEST:                                       │
│                                                                 │
│  1. HAPPY PATH                                                  │
│     ── Valid input → correct status code + correct body         │
│     ── Default values work                                      │
│     ── Response has expected shape                              │
│                                                                 │
│  2. VALIDATION ERRORS                                           │
│     ── Missing required fields → 422                            │
│     ── Fields violating constraints → 422                       │
│     ── Wrong types → 422                                        │
│     ── Boundary values (min, max, exactly-at-limit)             │
│                                                                 │
│  3. BUSINESS ERRORS                                             │
│     ── Resource not found → 404                                 │
│     ── Duplicate creation → 409 (if applicable)                 │
│     ── Invalid state transition → 400 (if applicable)           │
│                                                                 │
│  4. EDGE CASES                                                  │
│     ── Empty collections → returns []                           │
│     ── Special characters in input                              │
│     ── Very large valid input                                   │
│                                                                 │
│  USE:                                                           │
│  ── TestClient for all request/response tests (Parts 2-4)      │
│  ── dependency_overrides for isolation (Part 4)                 │
│  ── @pytest.mark.parametrize for validation coverage (Part 3)   │
│  ── Factories for test data (Part 5)                            │
│  ── conftest.py for shared fixtures (Part 5)                    │
│                                                                 │
│  TARGET: 80%+ code coverage for your Week 3-4 project.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.6 Testing Lifespan Events

**The `with TestClient(app)` context manager was mentioned in 2.5 as a way to trigger lifespan. Here is what that actually means and how to test it.**

**Modern FastAPI app with a lifespan hook:**

```python
# app.py extension
from contextlib import asynccontextmanager
from fastapi import FastAPI

# A resource that must be initialized before the app handles requests
class ConnectionPool:
    def __init__(self):
        self.ready = False

    async def connect(self):
        # Simulate opening connections
        self.ready = True

    async def disconnect(self):
        self.ready = False


connection_pool: ConnectionPool | None = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ── STARTUP ───────────────────────────────────────────────
    global connection_pool
    connection_pool = ConnectionPool()
    await connection_pool.connect()
    app.state.pool = connection_pool     # store on app.state

    yield                                # App is running and handling requests

    # ── SHUTDOWN ──────────────────────────────────────────────
    await connection_pool.disconnect()
    app.state.pool = None


app = FastAPI(lifespan=lifespan)


@app.get("/health")
async def health_check(request: Request) -> dict:
    pool = request.app.state.pool
    return {"status": "ok", "pool_ready": pool is not None and pool.ready}
```

**Tests:**

```python
# tests/test_lifespan.py
from fastapi.testclient import TestClient
from app import app


def test_startup_runs_when_context_manager_used():
    """With context manager form: startup must have run before first request."""
    with TestClient(app) as client:
        response = client.get("/health")
        assert response.status_code == 200
        assert response.json()["pool_ready"] is True


def test_startup_does_not_run_without_context_manager():
    """Without context manager: startup has NOT run → pool is None."""
    client = TestClient(app)   # No 'with' — no lifespan
    response = client.get("/health")
    # pool was never initialized → pool_ready is False
    assert response.json()["pool_ready"] is False


def test_shutdown_cleans_up_after_context_manager_exits():
    """After the with block, shutdown should have run."""
    with TestClient(app) as client:
        pass  # Do nothing — just trigger lifecycle

    # After exiting the with block, pool should be disconnected
    # (Test this via app.state if your endpoint exposes it)
    assert app.state.pool is None


def test_app_starts_without_raising():
    """The simplest lifespan test: startup should not throw."""
    try:
        with TestClient(app) as client:
            response = client.get("/health")
            assert response.status_code == 200
    except Exception as exc:
        pytest.fail(f"App startup raised an exception: {exc}")
```

```
┌─────────────────────────────────────────────────────────────────┐
│            THE LIFESPAN RULE — MEMORISE THIS                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TestClient(app)                                                │
│  ── Does NOT trigger startup/shutdown                           │
│  ── app.state is uninitialised                                  │
│  ── Fine for: endpoints with no startup dependencies            │
│                                                                 │
│  with TestClient(app) as client:                                │
│  ── startup() runs on __enter__                                 │
│  ── app.state is populated                                      │
│  ── shutdown() runs on __exit__                                 │
│  ── Required for: any endpoint that reads from app.state,       │
│       uses a DB pool, Redis client, or any resource             │
│       initialised in lifespan                                   │
│                                                                 │
│  When in doubt: use the context manager form.                   │
│  The only cost is a few milliseconds of startup time.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.7 Test Coverage Measurement

**Your project requires 80%+ test coverage. Here is how to install, run, and interpret the tool — and what the number actually means.**

**Setup:**

```toml
# pyproject.toml
[project.optional-dependencies]
test = [
    "pytest",
    "pytest-asyncio",
    "httpx",
    "respx",
    "pytest-cov",      # ← add this
]

[tool.coverage.run]
source = ["app", "storage", "models"]   # modules to measure
omit = [
    "tests/*",
    "*/migrations/*",
]

[tool.coverage.report]
show_missing = true
fail_under = 80
```

**Running coverage:**

```bash
# Terminal report — shows which lines are NOT covered
pytest --cov=app --cov-report=term-missing

# HTML report — click through in a browser (most useful)
pytest --cov=app --cov-report=html
# → open htmlcov/index.html

# Both at once
pytest --cov=app --cov-report=term-missing --cov-report=html

# Fail the CI run if coverage drops below 80%
pytest --cov=app --cov-fail-under=80
```

**Reading the output:**

```
---------- coverage: platform darwin, python 3.12 ----------
Name        Stmts   Miss  Cover  Missing
------------------------------------------
app.py         54     10    81%  23, 47-50, 68-72
storage.py     18      0   100%
models.py      12      2    83%  31, 32
------------------------------------------
TOTAL          84     12    86%
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 COVERAGE REPORT ANATOMY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  app.py      54    10    81%   23, 47-50, 68-72                 │
│   │           │     │     │    │                                │
│   │           │     │     │    └── Line numbers NEVER executed  │
│   │           │     │     └─────── % of lines that WERE run     │
│   │           │     └───────────── Lines not covered (count)    │
│   │           └─────────────────── Total executable lines       │
│   └─────────────────────────────── Module name                  │
│                                                                 │
│  "Missing: 47-50" means lines 47, 48, 49, 50 are a block       │
│  that no test executed. Open app.py and look at those lines.   │
│  They are likely inside an error handler or edge-case branch.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to act on the missing lines report:**

```python
# Suppose line 47 is this:
@app.exception_handler(TaskNotFoundError)
async def task_not_found_handler(request, exc):
    return JSONResponse(           # ← line 47: never executed
        status_code=404,
        content={"error": "not_found", "message": str(exc)}
    )

# The fix: write a test that TRIGGERS this handler
def test_custom_not_found_handler_is_invoked(client: TestClient):
    response = client.get("/tasks/99999")
    assert response.status_code == 404
    assert response.json()["error"] == "not_found"   # custom field
    # Now line 47 is covered
```

```
┌─────────────────────────────────────────────────────────────────┐
│         WHAT COVERAGE MEASURES — AND WHAT IT DOESN'T           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ MEASURES:                                                   │
│  ── Whether each line was executed at least once                │
│  ── Which branches of if/else were taken                        │
│     (with --cov-branch for branch coverage)                     │
│  ── Which functions were called                                 │
│                                                                 │
│  ❌ DOES NOT MEASURE:                                           │
│  ── Whether your assertions are CORRECT                         │
│  ── Whether edge cases are MEANINGFUL                           │
│  ── Whether you tested the right scenarios                      │
│                                                                 │
│  A test that calls every line but asserts nothing               │
│  scores 100% coverage and tests nothing.                        │
│                                                                 │
│  80% is a FLOOR, not a GOAL. Use missing-line reports           │
│  as a MAP to find untested code paths — then write              │
│  tests that assert real behavior on those paths.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You run pytest --cov=app --cov-report=term-missing and see:    │
│  app.py    54    8    85%    12, 38-41, 57                      │
│  Line 12 is the first line of a custom exception handler for    │
│  a DuplicateTaskError your app raises on POST /tasks when the   │
│  title already exists. Lines 38-41 are inside an               │
│  `if search and len(search) > 50` guard. Line 57 is a          │
│  `return []` inside a `try/except` block. Write the test        │
│  function names (you don't need to implement them) that would   │
│  cover each group of missing lines.                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION — test names that map to each missing block

# Line 12 — DuplicateTaskError handler
def test_duplicate_title_returns_409_with_custom_error_format():
    # POST /tasks with same title twice → triggers handler on line 12
    ...

# Lines 38-41 — search > 50 chars guard
def test_search_query_over_50_characters_is_handled():
    # GET /tasks?search=<51-char string> → enters the guard
    ...

# Line 57 — return [] inside except
def test_list_tasks_returns_empty_list_when_storage_raises():
    # Force storage.list_all() to raise → hits the except branch
    ...
```

---

## 5.8 Testing Background Tasks

**FastAPI's `BackgroundTasks` runs functions AFTER the response is sent to the client. Testing them requires understanding one critical fact about TestClient.**

**App code (background tasks):**

```python
# app.py extension
from fastapi import BackgroundTasks
import logging

logger = logging.getLogger(__name__)


def send_task_created_email(task_id: int, owner_email: str) -> None:
    """Send an email notification. Production: call email service here."""
    logger.info(f"Sending email to {owner_email} for task {task_id}")


def update_analytics_counter(task_id: int) -> None:
    """Increment analytics. Production: write to analytics DB."""
    logger.info(f"Analytics updated for task {task_id}")


@app.post("/tasks", status_code=201)
async def create_task(
    task: TaskCreate,
    background_tasks: BackgroundTasks,
    storage: TaskStorage = Depends(get_storage),
) -> TaskResponse:
    task_data = {**task.model_dump(), "completed": False}
    created = storage.create(task_data)

    # Schedule background work — runs after this response is sent
    background_tasks.add_task(
        send_task_created_email,
        task_id=created["id"],
        owner_email="owner@example.com"
    )
    background_tasks.add_task(update_analytics_counter, task_id=created["id"])

    return TaskResponse(**created)
```

**Tests:**

```python
# tests/test_background_tasks.py
from unittest.mock import patch, call


def test_creating_task_schedules_email(client: TestClient):
    """Creating a task should trigger the email notification function."""
    with patch("app.send_task_created_email") as mock_email:
        response = client.post("/tasks", json={"title": "Email me"})

        assert response.status_code == 201
        task_id = response.json()["id"]

        # TestClient executes background tasks synchronously —
        # by the time client.post() returns, the tasks have already run.
        mock_email.assert_called_once_with(
            task_id=task_id,
            owner_email="owner@example.com"
        )


def test_creating_task_schedules_analytics(client: TestClient):
    """Creating a task should trigger the analytics update."""
    with patch("app.update_analytics_counter") as mock_analytics:
        response = client.post("/tasks", json={"title": "Track me"})

        assert response.status_code == 201
        mock_analytics.assert_called_once_with(task_id=response.json()["id"])


def test_both_background_tasks_are_scheduled(client: TestClient):
    """Both background tasks must be scheduled on every create."""
    with (
        patch("app.send_task_created_email") as mock_email,
        patch("app.update_analytics_counter") as mock_analytics,
    ):
        client.post("/tasks", json={"title": "Both tasks"})

        mock_email.assert_called_once()
        mock_analytics.assert_called_once()


def test_background_task_failure_does_not_affect_http_response(client: TestClient):
    """A crashing background task should NOT cause a 500 response.
    
    This test DOCUMENTS the behavior. If your app lets background
    task errors propagate as HTTP errors, this test will catch that
    regression the moment you change the behavior.
    """
    with patch("app.send_task_created_email", side_effect=Exception("SMTP down")):
        response = client.post("/tasks", json={"title": "Failing email"})
        # The HTTP response was already sent before the task ran.
        # TestClient may raise or not depending on FastAPI's error handling.
        # Document whichever behavior your app actually has:
        assert response.status_code == 201


def test_no_background_tasks_on_failed_creation(client: TestClient):
    """If the endpoint fails (422), no background tasks should be scheduled."""
    with (
        patch("app.send_task_created_email") as mock_email,
        patch("app.update_analytics_counter") as mock_analytics,
    ):
        response = client.post("/tasks", json={"title": ""})  # Invalid: empty

        assert response.status_code == 422
        mock_email.assert_not_called()
        mock_analytics.assert_not_called()
```

```
┌─────────────────────────────────────────────────────────────────┐
│         TestClient AND BACKGROUND TASKS — THE KEY FACT          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IN PRODUCTION:                                                 │
│  1. Response is sent to client (201 Created)                    │
│  2. THEN background tasks run (client is already gone)          │
│                                                                 │
│  IN TestClient:                                                 │
│  1. Background tasks run SYNCHRONOUSLY within client.post()     │
│  2. Response object is returned AFTER tasks have completed      │
│                                                                 │
│  This means: mock_email.assert_called() works immediately       │
│  after client.post(). You don't need asyncio.sleep() or         │
│  any "wait for the task" logic.                                 │
│                                                                 │
│  ⚠️ THIS DIFFERS FROM PRODUCTION BEHAVIOR.                      │
│  In production, tasks run after the response is sent.           │
│  In TestClient, tasks run before you get the Response object.   │
│  Tests are still valid — the function is called either way.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your endpoint POST /users/register creates a user AND          │
│  schedules a background task `send_welcome_email(user_id, email)`│
│  Write a test that: (1) asserts 201 status, (2) verifies the    │
│  email function was called with the correct user_id (from the   │
│  response body) and the email that was sent in the request.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION

from unittest.mock import patch

def test_registration_sends_welcome_email(client: TestClient):
    with patch("app.send_welcome_email") as mock_email:
        response = client.post(
            "/users/register",
            json={"username": "alice", "email": "alice@example.com", "password": "secret"}
        )

        assert response.status_code == 201
        user_id = response.json()["id"]

        mock_email.assert_called_once_with(
            user_id=user_id,
            email="alice@example.com"
        )
```

---

## 5.9 Database Testing Preview (Bridge to Week 6)

**You're currently using an in-memory `TaskStorage` override. In Week 6, you'll swap it for a real PostgreSQL database via SQLAlchemy. The `dependency_overrides` pattern you learned today is EXACTLY what makes this transition work.**

```
┌─────────────────────────────────────────────────────────────────┐
│         THREE DATABASE TEST ISOLATION STRATEGIES                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STRATEGY 1: IN-MEMORY FAKE (current approach — Week 3-4)      │
│  ── Override get_storage() with fresh TaskStorage()             │
│  ── No PostgreSQL involved                                      │
│  ── Fastest, simplest                                           │
│  ── ✅ Correct for now                                          │
│  ── ❌ Doesn't test SQL queries, constraints, or migrations      │
│                                                                 │
│  STRATEGY 2: TRUNCATE TABLES BETWEEN TESTS (don't do this)     │
│  ── Use a real test database                                    │
│  ── DELETE FROM tasks; between tests                            │
│  ── Slow (DDL between every test)                               │
│  ── Dirty state if a test crashes mid-delete                    │
│  ── ❌ Fragile, slow, wrong                                     │
│                                                                 │
│  STRATEGY 3: TRANSACTION ROLLBACK (✅ Week 6 pattern)          │
│  ── Wrap each test in a database transaction                    │
│  ── Roll back EVERYTHING after the test, pass or fail           │
│  ── Database returns to clean state in microseconds             │
│  ── No deletions, no table truncation                           │
│  ── ✅ Fast, clean, correct                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Preview of the Week 6 pattern (do not implement yet — just read this):**

```python
# PREVIEW — Week 6 will implement this in full
# The key insight: dependency_overrides + yield fixture + rollback

from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine

TEST_DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/test_db"
engine = create_async_engine(TEST_DATABASE_URL)


@pytest.fixture
async def db_session():
    """Each test gets a transaction that ROLLS BACK after the test.
    No matter what the test does — inserts, updates, deletes —
    the database is restored to its exact state before the test ran.
    """
    async with AsyncSession(engine) as session:
        await session.begin()          # Open transaction
        yield session                  # Test runs inside this transaction
        await session.rollback()       # Undo EVERYTHING — clean state restored


@pytest.fixture
def client(db_session: AsyncSession) -> TestClient:
    """Inject the rollback session into the app via dependency_overrides.
    The app thinks it's using its normal DB session.
    It's actually using one that will roll back.
    """
    app.dependency_overrides[get_db] = lambda: db_session
    yield TestClient(app)
    app.dependency_overrides.clear()


# Your tests don't change at all:
def test_create_task(client: TestClient):
    response = client.post("/tasks", json={"title": "DB task"})
    assert response.status_code == 201
    # After this test: the DB row is GONE. Rolled back.
    # The next test starts from a clean database.
```

> "Look at how little changes. You still use `dependency_overrides`. You still have a `client` fixture. The only difference is what you inject — a rolling-back session instead of an in-memory class. That's the power of proper dependency injection."

**The connection between now and Week 6:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  TODAY (Weeks 3-4):                  WEEK 6:                    │
│                                                                 │
│  get_storage()                       get_db()                   │
│       │                                   │                     │
│       ▼                                   ▼                     │
│  TaskStorage()  ←── override ───▶  AsyncSession  ←── override  │
│  (in-memory)          │            (real DB, but  │             │
│                        │             rolls back)  │             │
│                        └────────────────────────────            │
│                           same dependency_overrides pattern      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.10 Debugging Failed Tests

**When a test fails, you need a systematic approach. Staring at the failure message is not an approach.**

**Reading a pytest failure:**

```
FAILED tests/test_tasks.py::test_create_task - AssertionError: assert 200 == 201

E   assert 200 == 201
E    +  where 200 = <Response [200 OK]>.status_code
```

```
┌─────────────────────────────────────────────────────────────────┐
│                ANATOMY OF A pytest FAILURE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FAILED tests/test_tasks.py::test_create_task                  │
│    ▲              ▲                   ▲                         │
│    │              └── file             └── function name        │
│    └── FAILED (vs PASSED, ERROR)                                │
│                                                                 │
│  AssertionError: assert 200 == 201                              │
│                         ▲      ▲                                │
│                         │      └── expected (right side)        │
│                         └──────── actual (left side)            │
│                                                                 │
│  E    +  where 200 = <Response [200 OK]>.status_code            │
│                        ▲                                        │
│                        pytest shows WHERE 200 came from         │
│                        (it was response.status_code)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Debugging flags — use these in order of invasiveness:**

```bash
# 1. Run only the failing test (stop noise from others)
pytest tests/test_tasks.py::test_create_task -v

# 2. Show print() output (suppressed by default)
pytest tests/test_tasks.py::test_create_task -v -s

# 3. Stop at the first failure — don't run the rest
pytest tests/test_tasks.py -x

# 4. Compact traceback — less clutter
pytest tests/test_tasks.py --tb=short

# 5. Full traceback with local variable values
pytest tests/test_tasks.py --tb=long

# 6. Drop into the debugger on failure
pytest tests/test_tasks.py --pdb
```

**Adding diagnostic output:**

```python
def test_create_task_returns_201(client: TestClient):
    response = client.post("/tasks", json={"title": "Debug me"})

    # Add these TEMPORARILY when debugging — remove before committing
    print(f"\nStatus:  {response.status_code}")
    print(f"Body:    {response.json()}")
    print(f"Headers: {dict(response.headers)}")

    assert response.status_code == 201
    # Run: pytest test_tasks.py::test_create_task -v -s
    # The -s flag makes print() visible in the terminal
```

**Using breakpoint() for interactive inspection:**

```python
def test_create_task_returns_201(client: TestClient):
    response = client.post("/tasks", json={"title": "Inspect me"})

    breakpoint()   # Execution pauses here — open interactive debugger

    assert response.status_code == 201

# In the pdb prompt:
# (Pdb) response.status_code    → 200  (the actual value)
# (Pdb) response.json()         → {"id": 1, "title": "Inspect me", ...}
# (Pdb) response.headers        → Headers({"content-type": "application/json"})
# (Pdb) c                       → continue (run remaining assertions)
# (Pdb) q                       → quit test run entirely
```

**Common failure patterns and their causes:**

```
┌─────────────────────────────────────────────────────────────────┐
│         COMMON FAILURE PATTERNS → ROOT CAUSES                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  assert 200 == 201                                              │
│  → Endpoint uses default status 200. Check:                     │
│    @app.post("/tasks", status_code=201)                         │
│                                                                 │
│  assert 422 == 201                                              │
│  → Pydantic rejected your request body. Check:                  │
│    print(response.json()) — read the validation errors           │
│    Your json={} payload likely violates a Field constraint       │
│                                                                 │
│  KeyError: 'id'                                                 │
│  → response.json() is an error body, not a task body. Check:    │
│    print(response.json()) BEFORE asserting fields               │
│                                                                 │
│  assert 2 == 1  (in test expecting id==1)                        │
│  → State leaked from a previous test. Fix:                      │
│    Use dependency_overrides with fresh storage per test          │
│                                                                 │
│  fixture 'client' not found                                     │
│  → conftest.py not in the right directory, or                   │
│    fixture scope mismatch. Check file paths.                    │
│                                                                 │
│  RuntimeError: Event loop is closed                             │
│  → asyncio_mode not set in pyproject.toml, or                   │
│    mixing sync and async fixtures incorrectly                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A test fails with: assert 422 == 201. Without adding print()   │
│  or a debugger, name THREE things you would check first — and   │
│  for each one, state which tool or command you'd use.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION

# 1. Read the 422 body — add print(response.json()) and run with -s
#    pytest test_tasks.py::my_test -v -s
#    The output shows which field failed validation and why.

# 2. Check your json={} payload against the Pydantic model.
#    Did you send {"title": ""} when min_length=1?
#    Did you omit a required field?

# 3. Check if the field type is wrong.
#    {"priority": "high"} → fails int coercion → 422
#    Run the test with --tb=long to see which line caused the assertion.
#    Use breakpoint() to inspect response.json()["detail"][0]["loc"]
#    to find exactly which field failed.
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                FASTAPI TESTING QUICK REFERENCE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SETUP:                                                         │
│      from fastapi.testclient import TestClient                  │
│      from app import app                                        │
│      client = TestClient(app)               # no lifespan       │
│      with TestClient(app) as client: ...    # with lifespan     │
│                                                                 │
│  MAKE REQUESTS:                                                 │
│      client.get("/path")                                        │
│      client.post("/path", json={...})       # application/json  │
│      client.post("/path", data={...})       # form-urlencoded   │
│      client.post("/path", files={"f": ("name", bytes, "mime")}) │
│      client.post("/path", files={...}, data={...}) # file+form  │
│      client.put("/path", json={...})                            │
│      client.patch("/path", json={...})                          │
│      client.delete("/path")                                     │
│      client.get("/path", params={"key": "val"})                 │
│      client.get("/path", headers={"X-Key": "val"})              │
│      client.get("/path", cookies={"session": "token"})          │
│      client.get("/path", follow_redirects=False) # test 307/308 │
│                                                                 │
│  CHECK RESPONSES:                                               │
│      response.status_code          # 200, 201, 404, 422...      │
│      response.json()               # dict or list               │
│      response.text                 # raw string body            │
│      response.headers              # response headers dict      │
│      response.headers["location"]  # redirect target            │
│      response.cookies              # cookies set by server      │
│      response.cookies["key"]       # specific cookie value      │
│                                                                 │
│  DEPENDENCY OVERRIDES:                                          │
│      app.dependency_overrides[original] = lambda: replacement   │
│      app.dependency_overrides[get_current_user] = lambda: user  │
│      app.dependency_overrides.clear()     # always clean up!    │
│                                                                 │
│  ASYNC TESTING:                                                 │
│      from httpx import AsyncClient, ASGITransport               │
│      transport = ASGITransport(app=app)                         │
│      async with AsyncClient(                                    │
│          transport=transport, base_url="http://test"            │
│      ) as client:                                               │
│          response = await client.get("/path")                   │
│                                                                 │
│  CONCURRENT TESTING:                                            │
│      responses = await asyncio.gather(                          │
│          *[async_client.post("/path", json={...})               │
│            for i in range(10)]                                  │
│      )                                                          │
│                                                                 │
│  MOCKING EXTERNAL HTTP CALLS:                                   │
│      import respx, httpx                                        │
│      @respx.mock                                                │
│      def test_x(client):                                        │
│          respx.get("https://api.example.com/...").mock(         │
│              return_value=httpx.Response(200, json={...})       │
│          )                                                      │
│                                                                 │
│  BACKGROUND TASKS:                                              │
│      from unittest.mock import patch                            │
│      with patch("app.my_task_fn") as mock_task:                 │
│          client.post("/endpoint", json={...})                   │
│          mock_task.assert_called_once_with(arg=value)           │
│                                                                 │
│  COVERAGE:                                                      │
│      pytest --cov=app --cov-report=term-missing                 │
│      pytest --cov=app --cov-report=html                         │
│      pytest --cov=app --cov-fail-under=80                       │
│                                                                 │
│  DEBUGGING:                                                     │
│      pytest test_tasks.py::test_x -v -s    # show print()      │
│      pytest test_tasks.py -x               # stop at first fail │
│      pytest test_tasks.py --pdb            # drop into debugger │
│      breakpoint()   # inside test — interactive pdb             │
│                                                                 │
│  PYTEST-ASYNCIO CONFIG (pyproject.toml):                        │
│      [tool.pytest.ini_options]                                  │
│      asyncio_mode = "auto"     # no @mark.asyncio needed        │
│      # asyncio_mode = "strict" # must mark each async test      │
│                                                                 │
│  COMMON ASSERTIONS:                                             │
│      assert response.status_code == 201                         │
│      assert response.json()["field"] == "value"                 │
│      assert "id" in response.json()                             │
│      assert isinstance(response.json(), list)                   │
│      assert len(response.json()["detail"]) > 0   # 422 errors  │
│      assert "x-request-id" in response.headers                  │
│      assert "refresh_token" in response.cookies                 │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ Only asserting body, not status_code                    │
│      ❌ Not clearing dependency_overrides after test             │
│      ❌ Sharing state between tests (no isolation)              │
│      ❌ Only testing the happy path (skipping 422, 404)         │
│      ❌ Using AsyncClient when TestClient is sufficient          │
│      ❌ Testing implementation details instead of HTTP contract │
│      ❌ Not overriding get_current_user for protected endpoints  │
│      ❌ Making real network calls in tests (use respx)          │
│      ❌ TestClient(app) instead of with TestClient(app)          │
│         when endpoint reads from app.state                      │
│      ❌ Treating 80% coverage as a ceiling instead of a floor   │
│      ❌ Missing asyncio_mode in pyproject.toml (async tests     │
│         silently never run)                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  API TESTING = TESTING THE HTTP CONTRACT                        │
│                                                                 │
│  Your clients don't call Python functions.                      │
│  They send HTTP requests. Test what they experience.            │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                   │
│  │ Request  │ ──▶│ FastAPI  │ ──▶│ Response │                   │
│  │ (JSON,   │    │ Pipeline │    │ (Status, │                   │
│  │  form,   │    │ (FULL)   │    │  JSON,   │                   │
│  │  files,  │    │          │    │  cookies,│                   │
│  │  cookies)│    │          │    │  headers)│                   │
│  └──────────┘    └──────────┘    └──────────┘                   │
│       ▲                               │                         │
│       │          TestClient           │                         │
│       └───────────────────────────────┘                         │
│                                                                 │
│                                                                 │
│  THE VENDING MACHINE ANALOGY — EXTENDED:                        │
│  ├─ TestClient      = testing panel on the back                 │
│  ├─ Request         = inserting coins + pressing a button        │
│  ├─ Response        = what drops out + what the display says     │
│  ├─ dep. overrides  = loading known test products                │
│  ├─ Auth override   = using a master key (bypass coin slot)      │
│  ├─ respx mock      = simulating the supplier without ordering   │
│  ├─ BackgroundTasks = the restocking crew (runs after you leave) │
│  └─ Coverage report = inspection checklist of every button       │
│       (tells you which buttons you forgot to press in testing)   │
│                                                                 │
│                                                                 │
│  SIX NON-NEGOTIABLE RULES:                                     │
│  1. Always assert status codes (not just body)                  │
│  2. Always test rejection (422, 404) — not just success         │
│  3. Always isolate state (dependency_overrides or fixtures)     │
│  4. Test BOTH auth and no-auth paths on protected endpoints     │
│  5. Never call real external APIs — mock them (respx/mock)      │
│  6. 80% coverage is a FLOOR, not a goal                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Connection to Upcoming Lectures

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 4, LECTURE 2 (next):                                      │
│  └─ API Design Principles (versioning, pagination, filtering)   │
│     You now know how to test paginated responses (§ 3.8).       │
│     Write tests for pagination AS YOU BUILD the endpoints.      │
│     Cursor pagination? Test the cursor token. API versioning?   │
│     Test that /v1/tasks and /v2/tasks behave differently.       │
│                                                                 │
│  WEEK 5-6 (Databases):                                          │
│  └─ dependency_overrides becomes CRITICAL.                      │
│     You'll override get_db() to inject a session that           │
│     ROLLS BACK after each test (§ 5.9 preview).                 │
│     Same pattern you learned today. Higher stakes.              │
│                                                                 │
│  WEEK 8 (External APIs + httpx):                                │
│  └─ respx full API: pattern matching, request body inspection,  │
│     async mocking, passthrough for selective real calls.        │
│     Basic respx is in § 4.8. Week 8 goes deeper.               │
│                                                                 │
│  WEEK 9 (Authentication):                                       │
│  └─ Real JWT encode/decode replaces the fake_user() pattern.    │
│     Override get_current_user() with a function that            │
│     decodes a real test-signed token. Full OAuth2 flow tests.   │
│                                                                 │
│  WEEK 11 (Celery Background Jobs):                              │
│  └─ BackgroundTasks testing (§ 5.8) is just the beginning.      │
│     Celery tasks run in a SEPARATE PROCESS. You'll need         │
│     a real broker in tests or the Celery task_always_eager      │
│     setting. Completely different challenge.                    │
│                                                                 │
│  WEEK 12 (WebSockets):                                          │
│  └─ AsyncClient becomes ESSENTIAL.                              │
│     WebSocket testing requires true async — TestClient          │
│     cannot simulate bidirectional real-time connections.        │
│                                                                 │
│  WEEK 15 (CI/CD + Observability):                               │
│  └─ pytest --cov-fail-under=80 runs in your GitHub Actions CI.   │
│     Coverage gates enforce your 80% floor automatically.        │
│     Structured logging tests verify correlation IDs propagate.  │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ Everything from today scales up.                            │
│     Unit tests (dep. overrides) + integration tests (real DB)   │
│     + end-to-end (full stack). 80%+ coverage measured in CI.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM FIRST, TOOL LAST                                       │
│  ────────────────────────                                       │
│  Students must FEEL why testing a function directly fails       │
│  before we hand them TestClient. Let them try and break it.     │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  API testing is abstract. We use a vending machine analogy      │
│  throughout. Every concept maps to something you can touch.     │
│                                                                 │
│  BUILD ON WHAT THEY KNOW                                        │
│  ───────────────────────                                        │
│  Week 2's pytest is the foundation. We are NOT teaching         │
│  testing from scratch. We are teaching a NEW CONTEXT            │
│  for skills they already have.                                  │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  pytest (Week 2)       → same assertions, fixtures, parametrize │
│  async/await (Week 1)  → endpoints are async; tests handle it  │
│  Pydantic (Week 3)     → validation errors become 422s         │
│  Depends() (Week 3)    → dependency_overrides is the payoff     │
│  Error handling (W1+3) → HTTPException becomes status codes     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                TESTING FASTAPI APPLICATIONS                     │
│                  (Reference Lecture — No Time Cap)              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (30 min)                                   │
│  ├─ 1.1 The Code Under Test                                     │
│  ├─ 1.2 The Naive Attempt (Demonstration)                       │
│  ├─ 1.3 What You're Actually Testing (The HTTP Contract)        │
│  └─ 1.4 The Vending Machine Analogy                             │
│                                                                 │
│  PART 2: THE TOOL                                               │
│  ├─ 2.1 Meet TestClient (The Testing Panel)                     │
│  ├─ 2.2 Your First API Test                                     │
│  ├─ 2.3 What TestClient Actually Does (Under the Hood)          │
│  ├─ 2.4 Every HTTP Method in TestClient          [MODIFIED]     │
│  ├─ 2.5 Building a Client Fixture (Connection to Week 2)        │
│  └─ 2.6 Cookies, File Uploads, Redirects,                       │
│          and Custom Response Headers             [NEW]          │
│                                                                 │
│  PART 3: TESTING WHAT MATTERS                                   │
│  ├─ 3.1 Testing Success Responses (The Happy Path)              │
│  ├─ 3.2 Testing Validation Errors (Pydantic Says No)            │
│  ├─ 3.3 Testing Not Found and Business Errors                   │
│  ├─ 3.4 Parametrize for Coverage (Connection to Week 2)         │
│  ├─ 3.5 Testing Response Shape (Trust, but Verify)              │
│  ├─ 3.6 Advanced Query Parameter Testing         [NEW]          │
│  ├─ 3.7 Testing Idempotency and Error Consistency [NEW]         │
│  └─ 3.8 Rate Limiting, Pagination, and OpenAPI   [NEW]          │
│                                                                 │
│  PART 4: ISOLATING YOUR CODE                                    │
│  ├─ 4.1 The Shared State Problem (Demonstration)                │
│  ├─ 4.2 dependency_overrides (FastAPI's Secret Weapon)          │
│  ├─ 4.3 Overriding Yield Dependencies                           │
│  ├─ 4.4 Unit vs Integration: When to Override, When Not To      │
│  ├─ 4.5 Overrides vs Mocks (Connection to Week 2)               │
│  ├─ 4.6 Testing Authenticated Endpoints          [NEW]          │
│  ├─ 4.7 Testing Middleware                       [NEW]          │
│  └─ 4.8 Mocking External HTTP Calls              [NEW]          │
│                                                                 │
│  PART 5: ASYNC TESTING & TEST ARCHITECTURE                      │
│  ├─ 5.1 When TestClient Isn't Enough                            │
│  ├─ 5.2 httpx.AsyncClient with ASGITransport     [MODIFIED]     │
│  ├─ 5.3 Test File Organization                                  │
│  ├─ 5.4 Factory Pattern for Test Data                           │
│  ├─ 5.5 Your Testing Strategy                                   │
│  ├─ 5.6 Testing Lifespan Events                  [NEW]          │
│  ├─ 5.7 Test Coverage Measurement                [NEW]          │
│  ├─ 5.8 Testing Background Tasks                 [NEW]          │
│  ├─ 5.9 Database Testing Preview (Bridge to Week 6) [NEW]       │
│  └─ 5.10 Debugging Failed Tests                  [NEW]          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Code Under Test

**Here's a simplified version of the Task Manager you're building. Keep it on screen — we'll refer to it constantly.**

```python
# app.py — Task Manager API (simplified from your Week 3 project)
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field

app = FastAPI()

# In-memory storage (no database yet — that's Week 5-6)
tasks_db: dict[int, dict] = {}
_id_counter: int = 0


class TaskCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    priority: int = Field(ge=1, le=5, default=3)


class TaskResponse(BaseModel):
    id: int
    title: str
    priority: int
    completed: bool


@app.post("/tasks", status_code=201)
async def create_task(task: TaskCreate) -> TaskResponse:
    global _id_counter
    _id_counter += 1
    task_data = {"id": _id_counter, **task.model_dump(), "completed": False}
    tasks_db[_id_counter] = task_data
    return TaskResponse(**task_data)


@app.get("/tasks/{task_id}")
async def get_task(task_id: int) -> TaskResponse:
    if task_id not in tasks_db:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")
    return TaskResponse(**tasks_db[task_id])


@app.get("/tasks")
async def list_tasks() -> list[TaskResponse]:
    return [TaskResponse(**t) for t in tasks_db.values()]
```

**Three endpoints. Simple CRUD. Now let's test them.**

---

## 1.2 The Naive Attempt

**In Week 2, you learned: call the function, check the result. Let's try that.**

```python
# test_naive.py — Attempt 1: treat it like a regular function
from app import create_task, TaskCreate

def test_create_task():
    task_input = TaskCreate(title="Write docs", priority=2)
    result = create_task(task_input)
    print(result)

test_create_task()
```

**Run it:**

```
<coroutine object create_task at 0x7f8b3c2a1e40>
RuntimeWarning: coroutine 'create_task' was never awaited
```

**Right — it's `async def`. You learned this in Week 1. Let's fix it.**

```python
# test_naive.py — Attempt 2: await the coroutine
import asyncio
from app import create_task, TaskCreate

async def test_create_task():
    task_input = TaskCreate(title="Write docs", priority=2)
    result = await create_task(task_input)
    
    print(type(result))     # <class 'app.TaskResponse'>
    print(result.title)     # "Write docs"
    print(result.priority)  # 2
    print(result.id)        # 1
    
    # Looks correct! Ship it!
    
    # ... but wait. One question:
    # What HTTP status code will the CLIENT receive?
    print(result.status_code)  # ❌ AttributeError!

asyncio.run(test_create_task())
```

**Now ask the class:**

> "The function returned a `TaskResponse` Python object. But your client doesn't receive Python objects. They receive HTTP responses with status codes, headers, and JSON bodies. Where is the status code? Where is the `Content-Type: application/json` header? Where is the 201 Created we specified in the decorator?"

Answer: **Nowhere. We bypassed the entire HTTP layer.**

---

**It gets worse. Try testing the error case:**

```python
# test_naive.py — Attempt 3: test a 404
import asyncio
from app import get_task

async def test_not_found():
    try:
        result = await get_task(task_id=999)
    except Exception as e:
        print(type(e))    # <class 'fastapi.exceptions.HTTPException'>
        print(e.status_code)  # 404
        print(e.detail)       # "Task 999 not found"
    
    # We caught a PYTHON EXCEPTION.
    # But clients don't catch Python exceptions.
    # Clients receive an HTTP response: {"detail": "Task 999 not found"}
    # with status code 404.
    #
    # If you have a custom exception handler (@app.exception_handler),
    # it was COMPLETELY BYPASSED.

asyncio.run(test_not_found())
```

**The insight:**

> "Calling the function directly is like testing a car engine on a workbench. You can verify the engine runs, but you have NO idea if the steering wheel turns the wheels, if the brake pedal stops the car, or if the speedometer shows the right number. You need to sit in the car and drive it."

---

## 1.3 What You're Actually Testing (The HTTP Contract)

**When a client talks to your API, this is the full pipeline:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE FULL REQUEST PIPELINE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT sends HTTP request                                      │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────┐                                                │
│  │ Middleware   │  ← Logging, CORS, timing                      │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Routing     │  ← Match URL to endpoint function              │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Path params │  ← Extract {task_id} from URL, coerce to int  │
│  │ + Query     │  ← Parse ?completed=true from query string     │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Pydantic    │  ← Validate JSON body → TaskCreate             │
│  │ Validation  │  ← REJECT invalid data with 422               │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Dependency  │  ← Resolve Depends() tree                      │
│  │ Injection   │                                                │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────────────────────────────────┐                    │
│  │  YOUR FUNCTION   ← You are testing HERE │                    │
│  └──────┬──────────────────────────────────┘                    │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Response    │  ← Serialize to JSON, set status code          │
│  │ Serializer  │  ← Apply response_model filtering              │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Exception   │  ← Convert HTTPException → proper response     │
│  │ Handlers    │                                                │
│  └──────┬──────┘                                                │
│         ▼                                                       │
│  CLIENT receives HTTP response                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When you call the function directly, you test ONE box. When you test through HTTP, you test ALL of them.**

```
┌─────────────────────────────────────────────────────────────────┐
│               DIRECT CALL vs HTTP TEST                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DIRECT CALL (await create_task(task)):                         │
│  ✅ Function logic                                               │
│  ❌ Middleware                                                   │
│  ❌ Routing                                                      │
│  ❌ Path/query parameter extraction                              │
│  ❌ Pydantic request validation                                  │
│  ❌ Dependency injection                                         │
│  ❌ Status code assignment                                       │
│  ❌ Response serialization                                       │
│  ❌ Exception handlers                                           │
│                                                                 │
│  HTTP TEST (client.post("/tasks", json={...})):                 │
│  ✅ Function logic                                               │
│  ✅ Middleware                                                    │
│  ✅ Routing                                                      │
│  ✅ Path/query parameter extraction                              │
│  ✅ Pydantic request validation                                  │
│  ✅ Dependency injection                                         │
│  ✅ Status code assignment                                       │
│  ✅ Response serialization                                       │
│  ✅ Exception handlers                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**This is the HTTP contract:**

> "The HTTP contract is the agreement between your API and its clients. It says: 'If you send THIS request, you will receive THIS response — with THIS status code, THIS body shape, and THIS error when something goes wrong.' Testing through HTTP tests the contract. Testing the function directly does not."

---

## 1.4 The Vending Machine Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE VENDING MACHINE ANALOGY                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your API is a VENDING MACHINE.                                 │
│                                                                 │
│  Customers interact with it through:                            │
│  ├─ BUTTONS (endpoints — /tasks, /tasks/{id})                   │
│  ├─ COIN SLOT (request body — the JSON payload)                 │
│  └─ DISPLAY (response — status code + JSON body)                │
│                                                                 │
│                                                                 │
│  TESTING THE FUNCTION DIRECTLY:                                 │
│  ──────────────────────────────                                 │
│  You open the back panel, reach in, and grab a snack.           │
│  It tells you the motor works. It tells you NOTHING about       │
│  whether a customer pressing B3 with $2.00 gets their chips.    │
│                                                                 │
│                                                                 │
│  TESTING THROUGH HTTP:                                          │
│  ─────────────────────                                          │
│  You walk up to the front, insert coins, press a button,        │
│  and check: Did the right snack drop? Did the display           │
│  show the right message? Did you get correct change?            │
│                                                                 │
│  THAT is testing the contract.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to testing:**

```
Vending Machine             │  FastAPI Testing
────────────────────────────│──────────────────────────────
The machine itself          │  Your FastAPI app
Buttons (A1, B3, C2)       │  Endpoints (POST /tasks)
Inserting coins             │  Sending a request body (JSON)
Display message             │  HTTP status code (201, 404, 422)
What drops into the slot    │  Response JSON body
Testing panel on the back   │  TestClient
Loading test products       │  Setting up test data / fixtures
Replacing the coin reader   │  dependency_overrides
  with a test one           │
```

---

# PART 2: THE TOOL

## 2.1 Meet TestClient (The Testing Panel)

**FastAPI provides `TestClient` — a way to send HTTP requests to your app WITHOUT starting a real server.**

```python
from fastapi.testclient import TestClient
from app import app

client = TestClient(app)
```

**That's it. One import, one line, and you have a fully functional HTTP client that talks directly to your app in-process.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    HOW TestClient WORKS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT TestClient (manual testing):                           │
│                                                                 │
│    Terminal 1:              Terminal 2:                          │
│    uvicorn app:app          curl http://localhost:8000/tasks     │
│    ↑ Real server running    ↑ Real HTTP over network            │
│                                                                 │
│                                                                 │
│  WITH TestClient (automated testing):                           │
│                                                                 │
│    Your test file:                                              │
│    client = TestClient(app)                                     │
│    client.get("/tasks")                                         │
│    ↑ No server. No network. Direct in-process call.             │
│      FastAPI's FULL pipeline runs, but no TCP/IP overhead.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In the vending machine analogy:**

> "TestClient is the testing panel built into the back of the machine. It lets you simulate button presses and coin insertions without being a real customer. But the machine's entire mechanism still runs — the motor turns, the belt moves, the snack drops. You're testing the real machine, just without the real customer."

---

## 2.2 Your First API Test

**Let's write a real test. Finally.**

```python
# test_tasks.py
from fastapi.testclient import TestClient
from app import app

client = TestClient(app)


def test_create_task():
    response = client.post(
        "/tasks",
        json={"title": "Write documentation", "priority": 2}
    )
    
    # Check the HTTP status code — we couldn't do this before!
    assert response.status_code == 201
    
    # Check the response body
    data = response.json()
    assert data["title"] == "Write documentation"
    assert data["priority"] == 2
    assert data["completed"] is False
    assert "id" in data
```

**Run it:**

```
$ pytest test_tasks.py -v

test_tasks.py::test_create_task PASSED
```

**Now ask the class:**

> "Compare this to the naive attempt in Part 1. What are we testing now that we weren't before?"

Answer:

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT WE'RE ACTUALLY TESTING NOW                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ The URL "/tasks" is correctly routed to create_task          │
│  ✅ The JSON body is parsed and validated by Pydantic            │
│  ✅ The status code is 201 (not default 200)                     │
│  ✅ The response body is serialized to JSON correctly            │
│  ✅ The response matches the TaskResponse schema                 │
│  ✅ The function logic creates the task with correct fields      │
│                                                                 │
│  All from ONE test. All impossible with a direct function call. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 What TestClient Actually Does (Under the Hood)

**You don't need to understand every detail, but the mental model matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                TestClient UNDER THE HOOD                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  client.post("/tasks", json={...})                              │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────┐                                       │
│  │  Build HTTP Request  │  Construct a real HTTP request object │
│  │  (method, URL, body, │  with proper headers, encoding, etc. │
│  │   headers)           │                                       │
│  └──────────┬───────────┘                                       │
│             ▼                                                    │
│  ┌──────────────────────┐                                       │
│  │  Send to ASGI app    │  Call your FastAPI app directly       │
│  │  (no network!)       │  through ASGI protocol (in-process)  │
│  └──────────┬───────────┘                                       │
│             ▼                                                    │
│  ┌──────────────────────┐                                       │
│  │  FastAPI processes   │  FULL pipeline: routing, validation, │
│  │  the request         │  dependencies, your function,        │
│  │                      │  serialization, exception handlers   │
│  └──────────┬───────────┘                                       │
│             ▼                                                    │
│  ┌──────────────────────┐                                       │
│  │  Return Response     │  Real HTTP response object with      │
│  │  object              │  status_code, headers, body          │
│  └──────────────────────┘                                       │
│                                                                 │
│  KEY POINT: TestClient is SYNCHRONOUS.                          │
│  It handles the async event loop internally.                    │
│  Your test functions are plain def, not async def.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why synchronous? Connection to Week 1 Lecture 3:**

> "Remember from the Async lecture: `asyncio.run()` starts the event loop. TestClient does that for you under the hood. You write normal `def test_...()` functions and let TestClient manage the async machinery. This is simpler for most tests — we'll cover async testing in Part 5 for when you need more control."

---

## 2.4 Every HTTP Method in TestClient

**TestClient mirrors the HTTP methods you learned in Week 3 Lecture 1:**

```python
from fastapi.testclient import TestClient
from app import app

client = TestClient(app)

# ── GET (retrieve) ─────────────────────────────────────
response = client.get("/tasks")
response = client.get("/tasks/42")
response = client.get("/tasks", params={"completed": "true"})

# ── POST (create) ──────────────────────────────────────
response = client.post("/tasks", json={"title": "New task"})

# ── PUT (full replace) ─────────────────────────────────
response = client.put("/tasks/42", json={"title": "Updated", "priority": 5})

# ── PATCH (partial update) ─────────────────────────────
response = client.patch("/tasks/42", json={"priority": 1})

# ── DELETE (remove) ────────────────────────────────────
response = client.delete("/tasks/42")
```

**Sending different types of data:**

```python
# JSON body (most common for APIs)
response = client.post("/tasks", json={"title": "Test"})

# Query parameters
response = client.get("/tasks", params={"completed": "true", "priority": "3"})

# Custom headers
response = client.get(
    "/tasks",
    headers={"X-Request-ID": "abc-123"}
)

# Path parameters — just build the URL string
task_id = 42
response = client.get(f"/tasks/{task_id}")
```

```python
# ── Form data (HTML form submissions, OAuth2 password grant) ───
response = client.post(
    "/auth/token",
    data={"username": "alice", "password": "secret"}
    # IMPORTANT: data={} sends application/x-www-form-urlencoded
    # This is NOT the same as json={}. Different Content-Type entirely.
)

# ── Single file upload (multipart/form-data) ───────────────────
response = client.post(
    "/upload",
    files={"file": ("report.pdf", b"PDF bytes here", "application/pdf")}
    #              ^ filename      ^ raw bytes         ^ MIME type
)

# ── File AND form fields in the same request ───────────────────
# (most real upload endpoints need both)
response = client.post(
    "/profile/photo",
    files={"photo": ("avatar.png", b"PNG bytes here", "image/png")},
    data={"alt_text": "My profile picture"}   # form field alongside file
)

# ── Multiple files under the same field name ───────────────────
response = client.post(
    "/upload/batch",
    files=[
        ("files", ("a.txt", b"content a", "text/plain")),
        ("files", ("b.txt", b"content b", "text/plain")),
    ]
)
```

```
┌─────────────────────────────────────────────────────────────────┐
│           json= vs data= vs files= — WHAT EACH SENDS           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  json={"key": "val"}                                            │
│  ── Content-Type: application/json                              │
│  ── Body: {"key": "val"}  (serialised JSON)                     │
│  ── Use for: REST APIs, Pydantic request bodies                 │
│                                                                 │
│  data={"key": "val"}                                            │
│  ── Content-Type: application/x-www-form-urlencoded             │
│  ── Body: key=val  (URL-encoded string)                         │
│  ── Use for: HTML forms, OAuth2 password flow                   │
│                                                                 │
│  files={"file": ("name", bytes, "mime")}                        │
│  ── Content-Type: multipart/form-data; boundary=...             │
│  ── Body: binary parts separated by a boundary string           │
│  ── Use for: file uploads                                       │
│                                                                 │
│  files + data together                                          │
│  ── Content-Type: multipart/form-data                           │
│  ── Body: file part + field parts in one multipart request      │
│  ── Use for: file upload with accompanying metadata             │
│                                                                 │
│  COMMON MISTAKE: sending a file with json={} instead of         │
│  files={}. You'll get 422 — Pydantic can't parse a JSON body   │
│  as an UploadFile. FastAPI knows the difference.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Reading the response:**

```python
response = client.post("/tasks", json={"title": "Test"})

# Status code (int)
response.status_code     # 201

# Body as JSON (dict or list)
response.json()          # {"id": 1, "title": "Test", ...}

# Body as raw text
response.text            # '{"id":1,"title":"Test",...}'

# Response headers
response.headers         # Headers({...})
response.headers["content-type"]  # "application/json"
```

---

## 2.5 Building a Client Fixture (Connection to Week 2)

**Connection to what you've learned:**

> "In Week 2, you learned `@pytest.fixture` for test setup. Let's use that to create a reusable client."

```python
# test_tasks.py
import pytest
from fastapi.testclient import TestClient
from app import app


@pytest.fixture
def client() -> TestClient:
    """Create a fresh TestClient for each test."""
    return TestClient(app)


def test_create_task(client: TestClient):
    response = client.post("/tasks", json={"title": "Test"})
    assert response.status_code == 201


def test_get_task(client: TestClient):
    # First create a task
    create_response = client.post("/tasks", json={"title": "Test"})
    task_id = create_response.json()["id"]
    
    # Then retrieve it
    response = client.get(f"/tasks/{task_id}")
    assert response.status_code == 200
    assert response.json()["title"] == "Test"
```

**Using TestClient as a context manager (connection to Week 1 Lecture 2):**

> "Remember context managers from Week 1? TestClient supports the `with` statement. This triggers FastAPI's startup and shutdown events (called 'lifespan'). For your current project this doesn't matter, but it will when your app has initialization logic."

```python
@pytest.fixture
def client() -> TestClient:
    # Triggers lifespan events (startup/shutdown)
    with TestClient(app) as c:
        yield c
```

```
┌─────────────────────────────────────────────────────────────────┐
│         TestClient: DIRECT vs CONTEXT MANAGER                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TestClient(app)                                                │
│  ── Simple instantiation                                        │
│  ── Does NOT trigger lifespan events                            │
│  ── Fine for apps with no startup/shutdown logic                │
│  ── ✅ Use this for your Week 3-4 project                       │
│                                                                 │
│  with TestClient(app) as client:                                │
│  ── Context manager form                                        │
│  ── DOES trigger lifespan events (startup → yield → shutdown)   │
│  ── Required when your app initializes resources on startup     │
│  ── ✅ Use this when you add database connections (Week 6)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.6 Cookies, File Uploads, Redirects, and Custom Response Headers

### Cookies

**Many APIs use cookies for sessions, refresh tokens, or CSRF protection. TestClient handles cookies exactly like a browser would.**

```python
# ── Sending cookies with a request ────────────────────────────
response = client.get(
    "/profile",
    cookies={"session_id": "abc-123"}
)

# ── Reading cookies the server sets on the response ───────────
response = client.post(
    "/auth/login",
    json={"username": "alice", "password": "secret"}
)
assert response.status_code == 200
assert "refresh_token" in response.cookies
assert response.cookies["refresh_token"] != ""

# ── Cookie jar: TestClient persists cookies between requests ──
# (when reusing the same client instance inside a with block)
with TestClient(app) as sticky_client:
    login_resp = sticky_client.post(
        "/auth/login",
        json={"username": "alice", "password": "secret"}
    )
    # The client now has the session cookie stored automatically
    profile_resp = sticky_client.get("/profile")  # Cookie sent automatically
    assert profile_resp.status_code == 200
```

**Testing that the server correctly sets a cookie:**

```python
# app.py extension:
from fastapi.responses import JSONResponse

@app.post("/auth/login")
async def login(credentials: LoginRequest):
    # validate credentials...
    response = JSONResponse(content={"message": "Logged in"})
    response.set_cookie(
        key="session_id",
        value="generated-session-token",
        httponly=True,
        secure=True,
        samesite="lax",
    )
    return response

# test:
def test_login_sets_session_cookie(client: TestClient):
    response = client.post(
        "/auth/login",
        json={"username": "alice", "password": "correct"}
    )
    assert response.status_code == 200

    # Verify the cookie exists and has the right security flags
    assert "session_id" in response.cookies

    # Inspect the raw Set-Cookie header for security attributes
    set_cookie_header = response.headers.get("set-cookie", "")
    assert "HttpOnly" in set_cookie_header
    assert "SameSite=lax" in set_cookie_header
```

---

### File Uploads and Downloads

**Extend the app with a simple upload endpoint:**

```python
# app.py extension
from fastapi import UploadFile, File

@app.post("/upload", status_code=201)
async def upload_file(file: UploadFile = File(...)) -> dict:
    contents = await file.read()
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents),
    }

@app.post("/upload/image", status_code=201)
async def upload_image(
    image: UploadFile = File(...),
    alt_text: str = Form(default=""),
) -> dict:
    if not image.content_type.startswith("image/"):
        raise HTTPException(
            status_code=400,
            detail=f"Expected an image, got '{image.content_type}'"
        )
    contents = await image.read()
    return {
        "filename": image.filename,
        "size": len(contents),
        "alt_text": alt_text,
    }
```

**Testing file upload endpoints:**

```python
def test_upload_file_returns_metadata(client: TestClient):
    """Uploaded file metadata should appear in the response."""
    response = client.post(
        "/upload",
        files={"file": ("report.pdf", b"PDF content here", "application/pdf")}
    )
    assert response.status_code == 201
    data = response.json()
    assert data["filename"] == "report.pdf"
    assert data["content_type"] == "application/pdf"
    assert data["size"] == len(b"PDF content here")


def test_upload_image_with_alt_text(client: TestClient):
    """Image upload with form field alongside file."""
    response = client.post(
        "/upload/image",
        files={"image": ("cat.png", b"\x89PNG...", "image/png")},
        data={"alt_text": "A cute cat"}
    )
    assert response.status_code == 201
    assert response.json()["alt_text"] == "A cute cat"


def test_upload_image_rejects_non_images(client: TestClient):
    """Non-image file should return 400 Bad Request."""
    response = client.post(
        "/upload/image",
        files={"image": ("data.csv", b"a,b,c", "text/csv")}
    )
    assert response.status_code == 400
    assert "image" in response.json()["detail"].lower()


def test_upload_empty_file(client: TestClient):
    """Empty file upload — document your chosen behavior."""
    response = client.post(
        "/upload",
        files={"file": ("empty.txt", b"", "text/plain")}
    )
    # Does your API accept empty files? 201 or 400?
    # This test DOCUMENTS that decision.
    assert response.status_code == 201
    assert response.json()["size"] == 0
```

**Testing file download endpoints:**

```python
# app.py extension
from fastapi.responses import StreamingResponse
import io

@app.get("/tasks/{task_id}/export")
async def export_task(task_id: int) -> StreamingResponse:
    task = tasks_db.get(task_id)
    if not task:
        raise HTTPException(status_code=404, detail="Task not found")
    csv_content = f"id,title,priority\n{task['id']},{task['title']},{task['priority']}\n"
    return StreamingResponse(
        io.StringIO(csv_content),
        media_type="text/csv",
        headers={"Content-Disposition": f"attachment; filename=task_{task_id}.csv"}
    )

# test:
def test_export_task_returns_csv(client: TestClient):
    """Export endpoint should return a downloadable CSV file."""
    create_resp = client.post("/tasks", json={"title": "Exported task", "priority": 2})
    task_id = create_resp.json()["id"]

    response = client.get(f"/tasks/{task_id}/export")

    assert response.status_code == 200
    assert response.headers["content-type"] == "text/csv; charset=utf-8"
    assert "attachment" in response.headers["content-disposition"]
    assert f"task_{task_id}.csv" in response.headers["content-disposition"]

    # Check the CSV body contains the right data
    lines = response.text.strip().split("\n")
    assert lines[0] == "id,title,priority"
    assert "Exported task" in lines[1]
```

---

### Redirects

**FastAPI uses `RedirectResponse` to send 307/308 status codes. TestClient follows redirects by default — you must disable that to test the redirect itself.**

```python
# app.py extension
from fastapi.responses import RedirectResponse

@app.get("/tasks/latest")
async def get_latest_task():
    """Redirect to the most recently created task."""
    if not tasks_db:
        raise HTTPException(status_code=404, detail="No tasks exist")
    latest_id = max(tasks_db.keys())
    return RedirectResponse(url=f"/tasks/{latest_id}", status_code=307)
```

```python
def test_redirect_status_code(client: TestClient):
    """GET /tasks/latest should redirect, not return data directly."""
    client.post("/tasks", json={"title": "Only task"})

    # follow_redirects=False: stop at the redirect, don't follow it
    response = client.get("/tasks/latest", follow_redirects=False)

    assert response.status_code == 307
    assert response.headers["location"] == "/tasks/1"


def test_redirect_is_followed_by_default(client: TestClient):
    """Without follow_redirects=False, TestClient follows the redirect."""
    client.post("/tasks", json={"title": "Target task"})

    response = client.get("/tasks/latest")   # follows redirect by default

    assert response.status_code == 200       # final destination's status
    assert response.json()["title"] == "Target task"


def test_redirect_on_empty_db_returns_404(client: TestClient):
    """If no tasks exist, the redirect endpoint should return 404."""
    response = client.get("/tasks/latest", follow_redirects=False)
    assert response.status_code == 404
```

```
┌─────────────────────────────────────────────────────────────────┐
│              REDIRECT STATUS CODES — QUICK GUIDE                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  301 Moved Permanently — method may change to GET on redirect   │
│  302 Found            — same; method may change                 │
│  303 See Other        — explicitly tells client to use GET      │
│  307 Temporary        — method PRESERVED (POST stays POST)      │
│  308 Permanent        — method PRESERVED (POST stays POST)      │
│                                                                 │
│  For API redirects: prefer 307 (preserve method) or 308.       │
│  For form redirects after POST: use 303 (redirect to GET).     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Custom Response Headers

**Testing that your app sets the right response headers:**

```python
def test_post_response_includes_location_header(client: TestClient):
    """REST convention: POST that creates a resource should include Location."""
    response = client.post("/tasks", json={"title": "New task"})
    assert response.status_code == 201
    # If your app follows REST conventions:
    # assert "location" in response.headers
    # assert response.headers["location"].startswith("/tasks/")

def test_content_type_is_json(client: TestClient):
    """All endpoints should return application/json Content-Type."""
    response = client.get("/tasks")
    assert "application/json" in response.headers["content-type"]
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your endpoint POST /documents accepts a PDF file (field name   │
│  "document") and an optional form field "title" (string).       │
│  Write THREE tests: (1) a successful upload with title,         │
│  (2) a rejection when a non-PDF is uploaded, and (3) verify     │
│  the response contains the filename and size.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION

def test_pdf_upload_with_title(client: TestClient):
    response = client.post(
        "/documents",
        files={"document": ("report.pdf", b"%PDF-1.4 ...", "application/pdf")},
        data={"title": "Q4 Report"}
    )
    assert response.status_code == 201
    assert response.json()["filename"] == "report.pdf"
    assert response.json()["title"] == "Q4 Report"


def test_non_pdf_upload_rejected(client: TestClient):
    response = client.post(
        "/documents",
        files={"document": ("image.png", b"\x89PNG...", "image/png")}
    )
    assert response.status_code == 400
    assert "pdf" in response.json()["detail"].lower()


def test_upload_response_contains_filename_and_size(client: TestClient):
    content = b"Hello PDF"
    response = client.post(
        "/documents",
        files={"document": ("doc.pdf", content, "application/pdf")}
    )
    assert response.status_code == 201
    data = response.json()
    assert "filename" in data
    assert data["size"] == len(content)
```

---

# PART 3: TESTING WHAT MATTERS

## 3.1 Testing Success Responses (The Happy Path)

**The happy path is the scenario where everything goes right. Start here, but don't stop here.**

```python
def test_create_task_returns_201(client: TestClient):
    """POST /tasks with valid data should return 201 Created."""
    response = client.post(
        "/tasks",
        json={"title": "Learn testing", "priority": 4}
    )
    
    assert response.status_code == 201  # Not 200 — the specific code matters
    
    data = response.json()
    assert data["title"] == "Learn testing"
    assert data["priority"] == 4
    assert data["completed"] is False  # Default value
    assert isinstance(data["id"], int)  # ID was assigned


def test_create_task_with_defaults(client: TestClient):
    """POST /tasks with only required fields uses defaults."""
    response = client.post("/tasks", json={"title": "Minimal task"})
    
    assert response.status_code == 201
    assert response.json()["priority"] == 3  # Default from Pydantic model


def test_get_existing_task(client: TestClient):
    """GET /tasks/{id} for an existing task returns 200."""
    # Arrange: create a task first
    create_resp = client.post("/tasks", json={"title": "Find me"})
    task_id = create_resp.json()["id"]
    
    # Act: retrieve it
    response = client.get(f"/tasks/{task_id}")
    
    # Assert
    assert response.status_code == 200
    assert response.json()["title"] == "Find me"
    assert response.json()["id"] == task_id
```

**Notice the Arrange-Act-Assert pattern (from Week 2):**

```
┌─────────────────────────────────────────────────────────────────┐
│                   ARRANGE-ACT-ASSERT                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ARRANGE: Set up the preconditions                              │
│  ── Create a task so we have something to GET                   │
│                                                                 │
│  ACT: Perform the action being tested                           │
│  ── Send the GET request                                        │
│                                                                 │
│  ASSERT: Check the outcome                                      │
│  ── Status code is 200                                          │
│  ── Response body matches expectations                          │
│                                                                 │
│  This is the same pattern from Week 2,                          │
│  applied to HTTP requests instead of function calls.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Testing Validation Errors (Pydantic Says No)

**Connection to Week 3 Lecture 3:**

> "Remember Pydantic's field constraints? `Field(min_length=1, max_length=100)` and `Field(ge=1, le=5)` in your TaskCreate model. When a client sends data that violates those constraints, FastAPI returns 422 Unprocessable Entity — automatically. You MUST test this."

```python
def test_create_task_empty_title_returns_422(client: TestClient):
    """POST /tasks with empty title should be rejected."""
    response = client.post("/tasks", json={"title": "", "priority": 3})
    
    assert response.status_code == 422  # Unprocessable Entity
```

**But what does the 422 response BODY look like?**

```python
def test_create_task_empty_title_error_detail(client: TestClient):
    """422 response should include structured error details."""
    response = client.post("/tasks", json={"title": ""})
    
    assert response.status_code == 422
    
    # FastAPI returns a structured error body:
    error_body = response.json()
    print(error_body)
```

**Output:**

```json
{
    "detail": [
        {
            "type": "string_too_short",
            "loc": ["body", "title"],
            "msg": "String should have at least 1 character",
            "input": "",
            "ctx": {"min_length": 1}
        }
    ]
}
```

**Anatomy of a 422 error:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   422 ERROR ANATOMY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  {                                                              │
│    "detail": [                 ← List of errors (can be many)   │
│      {                                                          │
│        "type": "string_too_short",  ← Machine-readable type    │
│        "loc": ["body", "title"],    ← WHERE the error is       │
│                  ▲       ▲                                      │
│                  │       └── Field name                         │
│                  └────────── In the request body                │
│        "msg": "String should...",   ← Human-readable message   │
│        "input": "",                 ← What they actually sent   │
│        "ctx": {"min_length": 1}     ← Constraint that failed   │
│      }                                                          │
│    ]                                                            │
│  }                                                              │
│                                                                 │
│  "loc" values: ["body", "field"]    ← JSON body field           │
│                ["path", "task_id"]  ← Path parameter            │
│                ["query", "page"]    ← Query parameter           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Asserting specific error details:**

```python
def test_create_task_empty_title_error_location(client: TestClient):
    """Error should point to the correct field."""
    response = client.post("/tasks", json={"title": ""})
    
    errors = response.json()["detail"]
    assert len(errors) == 1
    assert errors[0]["loc"] == ["body", "title"]
    assert errors[0]["type"] == "string_too_short"
```

**Testing path parameter coercion:**

```python
def test_get_task_invalid_id_type(client: TestClient):
    """GET /tasks/abc should return 422 — 'abc' is not an int."""
    response = client.get("/tasks/abc")
    
    assert response.status_code == 422
    errors = response.json()["detail"]
    assert errors[0]["loc"] == ["path", "task_id"]
```

> "FastAPI declared `task_id: int` in the path. When a client sends `/tasks/abc`, FastAPI can't coerce `'abc'` to an integer and returns 422. This validation is part of the HTTP contract — test it."

---

## 3.3 Testing Not Found and Business Errors

**These test the error paths your application defines explicitly.**

```python
def test_get_nonexistent_task_returns_404(client: TestClient):
    """GET /tasks/{id} for a missing task should return 404."""
    response = client.get("/tasks/99999")
    
    assert response.status_code == 404
    assert response.json()["detail"] == "Task 99999 not found"


def test_404_response_structure(client: TestClient):
    """404 response should have the standard error format."""
    response = client.get("/tasks/99999")
    
    assert response.status_code == 404
    data = response.json()
    assert "detail" in data
    assert isinstance(data["detail"], str)
```

**Connection to Week 3 Lecture 4 — Custom exception handlers:**

> "If your app defines a custom exception handler with `@app.exception_handler`, testing through TestClient is the ONLY way to verify it works. A direct function call would raise the raw exception and skip your handler entirely."

```python
# In your app (from Week 3 Lecture 4):
class TaskNotFoundError(Exception):
    def __init__(self, task_id: int):
        self.task_id = task_id

@app.exception_handler(TaskNotFoundError)
async def task_not_found_handler(request, exc: TaskNotFoundError):
    return JSONResponse(
        status_code=404,
        content={
            "error": "not_found",
            "message": f"Task {exc.task_id} does not exist",
        }
    )

# Test that the custom handler formats the error correctly:
def test_custom_not_found_format(client: TestClient):
    response = client.get("/tasks/99999")
    
    assert response.status_code == 404
    data = response.json()
    assert data["error"] == "not_found"      # Custom field
    assert "99999" in data["message"]         # Includes the ID
```

---

## 3.4 Parametrize for Coverage (Connection to Week 2)

**Connection to what you've learned:**

> "In Week 2, you learned `@pytest.mark.parametrize` for running the same test with multiple inputs. This is PERFECT for validation testing — one test function, many invalid inputs."

```python
import pytest
from fastapi.testclient import TestClient
from app import app


@pytest.fixture
def client() -> TestClient:
    return TestClient(app)


@pytest.mark.parametrize("payload,expected_status", [
    # ── Happy path ──
    ({"title": "Valid task"}, 201),
    ({"title": "Valid", "priority": 1}, 201),
    ({"title": "Valid", "priority": 5}, 201),
    
    # ── Validation failures (422) ──
    ({"title": ""}, 422),                    # Too short (min_length=1)
    ({"title": "x" * 101}, 422),             # Too long (max_length=100)
    ({"title": "Valid", "priority": 0}, 422), # Priority too low (ge=1)
    ({"title": "Valid", "priority": 6}, 422), # Priority too high (le=5)
    ({}, 422),                                # Missing required field
])
def test_create_task_validation(
    client: TestClient,
    payload: dict,
    expected_status: int,
):
    response = client.post("/tasks", json=payload)
    assert response.status_code == expected_status
```

**Run it:**

```
$ pytest test_tasks.py::test_create_task_validation -v

test_create_task_validation[payload0-201] PASSED
test_create_task_validation[payload1-201] PASSED
test_create_task_validation[payload2-201] PASSED
test_create_task_validation[payload3-422] PASSED
test_create_task_validation[payload4-422] PASSED
test_create_task_validation[payload5-422] PASSED
test_create_task_validation[payload6-422] PASSED
test_create_task_validation[payload7-422] PASSED

8 passed
```

**8 scenarios. 1 test function. Every boundary of your Pydantic constraints verified.**

> "In the vending machine analogy: you're testing every coin denomination — valid coins, slugs, foreign coins, no coins. One testing procedure, many inputs."

**Make test IDs readable with `pytest.param`:**

```python
@pytest.mark.parametrize("payload,expected_status", [
    pytest.param(
        {"title": "Valid task"}, 201,
        id="valid-task-returns-201"
    ),
    pytest.param(
        {"title": ""}, 422,
        id="empty-title-rejected"
    ),
    pytest.param(
        {"title": "x" * 101}, 422,
        id="title-over-100-chars-rejected"
    ),
    pytest.param(
        {"title": "Valid", "priority": 0}, 422,
        id="priority-below-1-rejected"
    ),
])
def test_create_task_validation(client, payload, expected_status):
    response = client.post("/tasks", json=payload)
    assert response.status_code == expected_status
```

```
$ pytest -v

test_create_task_validation[valid-task-returns-201] PASSED
test_create_task_validation[empty-title-rejected] PASSED
test_create_task_validation[title-over-100-chars-rejected] PASSED
test_create_task_validation[priority-below-1-rejected] PASSED
```

---

## 3.5 Testing Response Shape (Trust, but Verify)

**It's not enough that the response returns 200. The body must have the right SHAPE.**

```python
def test_create_task_response_has_all_fields(client: TestClient):
    """Response must contain exactly the fields defined in TaskResponse."""
    response = client.post(
        "/tasks",
        json={"title": "Shape test", "priority": 2}
    )
    data = response.json()
    
    # Check ALL expected fields exist
    expected_fields = {"id", "title", "priority", "completed"}
    assert set(data.keys()) == expected_fields
    
    # Check field types
    assert isinstance(data["id"], int)
    assert isinstance(data["title"], str)
    assert isinstance(data["priority"], int)
    assert isinstance(data["completed"], bool)


def test_list_tasks_returns_array(client: TestClient):
    """GET /tasks should always return a JSON array."""
    response = client.get("/tasks")
    
    assert response.status_code == 200
    assert isinstance(response.json(), list)
```

**Why test the shape? Because clients depend on it.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHY TEST RESPONSE SHAPE?                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your frontend or mobile app expects:                           │
│  {                                                              │
│    "id": <int>,                                                 │
│    "title": <string>,                                           │
│    "priority": <int>,                                           │
│    "completed": <boolean>                                       │
│  }                                                              │
│                                                                 │
│  If you accidentally add, remove, or rename a field,            │
│  the frontend BREAKS. Your test catches this BEFORE deploy.     │
│                                                                 │
│  This is why response_model exists in FastAPI —                 │
│  but testing CONFIRMS it does what you think.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.6 Advanced Query Parameter Testing

**The one-line example in 2.4 showed how to send query parameters. This section is about how to ASSERT their behavior — required vs optional, type coercion, combinations, and edge cases.**

**Extend the app with a filterable list endpoint:**

```python
# app.py extension
from fastapi import Query

@app.get("/tasks")
async def list_tasks(
    completed: bool | None = None,           # Optional boolean filter
    priority: int | None = Query(            # Optional int, constrained
        default=None, ge=1, le=5
    ),
    search: str | None = None,               # Optional text filter
    size: int = Query(default=20, le=100),   # Capped at 100
    storage: TaskStorage = Depends(get_storage),
) -> list[TaskResponse]:
    tasks = storage.list_all()
    if completed is not None:
        tasks = [t for t in tasks if t["completed"] == completed]
    if priority is not None:
        tasks = [t for t in tasks if t["priority"] == priority]
    if search:
        tasks = [t for t in tasks if search.lower() in t["title"].lower()]
    return [TaskResponse(**t) for t in tasks[:size]]
```

**Testing optional vs required parameters:**

```python
def test_list_tasks_with_no_params_returns_all(client: TestClient):
    """All query params are optional — omitting them returns everything."""
    client.post("/tasks", json={"title": "Task A", "priority": 1})
    client.post("/tasks", json={"title": "Task B", "priority": 5})

    response = client.get("/tasks")   # No params at all
    assert response.status_code == 200
    assert len(response.json()) == 2


def test_filter_by_completed_true(client: TestClient):
    """completed=true should return only completed tasks."""
    client.post("/tasks", json={"title": "Done task"})
    # Assume PATCH to mark it complete (or pre-populate via storage fixture)
    response = client.get("/tasks", params={"completed": "true"})
    assert response.status_code == 200
    assert all(t["completed"] is True for t in response.json())


def test_filter_by_completed_false(client: TestClient):
    """completed=false should return only incomplete tasks."""
    client.post("/tasks", json={"title": "Pending task"})
    response = client.get("/tasks", params={"completed": "false"})
    assert all(t["completed"] is False for t in response.json())


def test_filter_by_priority(client: TestClient):
    """priority filter should only return matching tasks."""
    client.post("/tasks", json={"title": "Urgent", "priority": 5})
    client.post("/tasks", json={"title": "Low prio", "priority": 1})

    response = client.get("/tasks", params={"priority": "5"})
    data = response.json()
    assert len(data) == 1
    assert data[0]["priority"] == 5
```

**Testing query parameter type coercion — FastAPI coerces automatically, and REJECTS wrong types:**

```python
def test_invalid_priority_type_returns_422(client: TestClient):
    """priority=high cannot be coerced to int → 422."""
    response = client.get("/tasks", params={"priority": "high"})
    assert response.status_code == 422
    errors = response.json()["detail"]
    # Error should point to the query parameter, not the body
    assert any(e["loc"] == ["query", "priority"] for e in errors)


def test_priority_above_max_returns_422(client: TestClient):
    """priority=6 violates ge=1, le=5 → 422."""
    response = client.get("/tasks", params={"priority": "6"})
    assert response.status_code == 422


def test_size_above_limit_returns_422(client: TestClient):
    """size has le=100. Sending 101 → 422."""
    response = client.get("/tasks", params={"size": "101"})
    assert response.status_code == 422
    errors = response.json()["detail"]
    assert any(e["loc"] == ["query", "size"] for e in errors)
```

**Testing filter combinations (AND logic):**

```python
def test_combined_filters_are_anded(client: TestClient):
    """Multiple filters narrow results — they combine with AND."""
    client.post("/tasks", json={"title": "Match both", "priority": 5})
    client.post("/tasks", json={"title": "Priority only", "priority": 5})
    # "Match both" would be marked complete in a real test
    # This demonstrates the pattern — adapt to your storage

    response = client.get(
        "/tasks",
        params={"priority": "5"}
    )
    data = response.json()
    # All returned tasks match the priority filter
    assert all(t["priority"] == 5 for t in data)


def test_search_is_case_insensitive(client: TestClient):
    """Text search should match regardless of case."""
    client.post("/tasks", json={"title": "Write Documentation"})

    response = client.get("/tasks", params={"search": "documentation"})
    assert response.status_code == 200
    assert len(response.json()) == 1
    assert "Documentation" in response.json()[0]["title"]


def test_search_no_match_returns_empty_list(client: TestClient):
    """Search with no matches should return [] not 404."""
    client.post("/tasks", json={"title": "Existing task"})

    response = client.get("/tasks", params={"search": "xyzzy"})
    assert response.status_code == 200
    assert response.json() == []
```

> "Notice: no match → `[]` with status 200. Never return 404 for an empty collection. A collection that happens to be empty is still a valid, successful response. 404 means the COLLECTION ITSELF doesn't exist."

---

## 3.7 Testing Idempotency and Error Response Consistency

### Testing Idempotency

**The HTTP spec defines idempotency: calling the same operation multiple times produces the same result as calling it once. PUT and DELETE must be idempotent. Test it — don't assume it.**

```python
def test_delete_is_idempotent(client: TestClient):
    """Deleting the same resource twice should produce consistent behavior."""
    create_resp = client.post("/tasks", json={"title": "Delete me"})
    task_id = create_resp.json()["id"]

    first_delete = client.delete(f"/tasks/{task_id}")
    assert first_delete.status_code == 204   # Success: deleted

    second_delete = client.delete(f"/tasks/{task_id}")
    assert second_delete.status_code == 404  # Resource is gone — consistent 404


def test_put_is_idempotent(client: TestClient):
    """Calling PUT twice with the same payload should yield identical results."""
    create_resp = client.post("/tasks", json={"title": "Original", "priority": 3})
    task_id = create_resp.json()["id"]

    update_payload = {"title": "Updated Title", "priority": 5, "completed": False}

    first_put = client.put(f"/tasks/{task_id}", json=update_payload)
    second_put = client.put(f"/tasks/{task_id}", json=update_payload)

    assert first_put.status_code == 200
    assert second_put.status_code == 200
    assert first_put.json() == second_put.json()   # Identical results
```

> "Why does DELETE return 404 on the second call? Some argue it should return 204 again — the outcome is the same: the resource doesn't exist. Both are valid. What matters is that you DOCUMENT the behavior with a test. The test is your spec."

---

### Testing Error Response Consistency

**All your error responses should follow the same format. Clients parse your errors programmatically — inconsistent formats force them to write special-case code. Test the contract.**

```python
def test_all_error_responses_have_detail_key(client: TestClient):
    """Every 4xx response must contain a 'detail' field."""
    scenarios = [
        ("get",    "/tasks/99999",                 None),          # 404
        ("post",   "/tasks",                        None),          # 422 (no body)
        ("get",    "/tasks/abc",                    None),          # 422 (bad type)
        ("delete", "/tasks/99999",                  None),          # 404
    ]
    for method, url, body in scenarios:
        fn = getattr(client, method)
        response = fn(url) if body is None else fn(url, json=body)

        assert response.status_code >= 400, f"Expected error for {method.upper()} {url}"
        body_data = response.json()
        assert "detail" in body_data, (
            f"{method.upper()} {url} → {response.status_code} "
            f"but response missing 'detail' key. Got: {body_data}"
        )


def test_404_detail_is_string(client: TestClient):
    """HTTPException detail should be a plain string — not a nested object."""
    response = client.get("/tasks/99999")
    assert response.status_code == 404
    assert isinstance(response.json()["detail"], str)


def test_422_detail_is_list(client: TestClient):
    """Pydantic 422 detail is always a list of error objects."""
    response = client.post("/tasks", json={"title": ""})
    assert response.status_code == 422
    assert isinstance(response.json()["detail"], list)
    assert len(response.json()["detail"]) >= 1
```

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY ERROR CONSISTENCY MATTERS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FastAPI's DEFAULT error format:                                │
│  ├─ HTTPException (404, 403...)  → {"detail": "string"}         │
│  └─ Validation (422)            → {"detail": [{loc, msg, ...}]} │
│                                                                 │
│  If you use a custom exception handler that returns a           │
│  DIFFERENT shape — e.g., {"error": "not_found"} — your         │
│  frontend now has to check both formats. That's a bug.          │
│                                                                 │
│  The test above catches this automatically. If you add a        │
│  custom handler that breaks the shape, the test fails.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your app has POST /tasks (create) and DELETE /tasks/{id}.      │
│  Write a test that creates a task, deletes it, then verifies    │
│  that a GET on the deleted task returns 404 — and that a        │
│  second DELETE on the same ID also returns 404, not 204.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION

def test_deleted_task_is_truly_gone(client: TestClient):
    # Arrange
    create_resp = client.post("/tasks", json={"title": "Temporary"})
    assert create_resp.status_code == 201
    task_id = create_resp.json()["id"]

    # Act — delete it
    delete_resp = client.delete(f"/tasks/{task_id}")
    assert delete_resp.status_code == 204

    # Assert — GET confirms it's gone
    get_resp = client.get(f"/tasks/{task_id}")
    assert get_resp.status_code == 404

    # Assert — second DELETE is consistent (idempotent behavior)
    second_delete = client.delete(f"/tasks/{task_id}")
    assert second_delete.status_code == 404
```

---

## 3.8 Rate Limiting, Pagination Preview, and OpenAPI Schema

### Testing Rate Limiting (429 Responses)

**Rate limiters return 429 Too Many Requests when a client sends too many requests in a window. You need to test that your limiter fires correctly — and that it includes the `Retry-After` header.**

```python
# This pattern assumes slowapi or a custom rate limiter is configured.
# The override strategy: override the rate limit key function or
# set the limit very low in the test configuration.

def test_rate_limited_request_returns_429(client: TestClient):
    """Exceeding the rate limit should return 429, not 200."""
    # Set limit to 3/minute in test config — send 4 requests
    for i in range(3):
        response = client.get("/tasks")
        assert response.status_code == 200

    # 4th request should be rejected
    response = client.get("/tasks")
    assert response.status_code == 429


def test_rate_limit_response_has_retry_after(client: TestClient):
    """429 response should tell clients when to retry."""
    for _ in range(3):
        client.get("/tasks")

    response = client.get("/tasks")
    assert response.status_code == 429
    # Retry-After tells the client how many seconds to wait
    assert "retry-after" in response.headers
    assert int(response.headers["retry-after"]) > 0
```

> "Rate limiting testing is integration-heavy — you need the real limiter configured. In unit tests, you can override the rate limiter dependency to return a fake that always allows or always rejects. The important thing is to TEST BOTH paths: allowed and rejected."

---

### Pagination Preview

**Pagination is covered in depth in Week 4, Lecture 2. But you need to know NOW what to test so you can write tests alongside your implementation.**

```python
# Assume your endpoint returns paginated data in this shape:
# {
#   "items": [...],
#   "total": 100,
#   "page": 1,
#   "size": 20,
#   "pages": 5
# }

def test_paginated_response_has_required_keys(client: TestClient):
    """Pagination metadata must be present on every paginated response."""
    response = client.get("/tasks", params={"page": 1, "size": 10})
    assert response.status_code == 200
    data = response.json()

    required_keys = {"items", "total", "page", "size"}
    assert required_keys.issubset(data.keys()), (
        f"Missing pagination keys: {required_keys - data.keys()}"
    )


def test_pagination_respects_size_param(client: TestClient):
    """size param should limit the number of items returned."""
    for i in range(5):
        client.post("/tasks", json={"title": f"Task {i}"})

    response = client.get("/tasks", params={"page": 1, "size": 2})
    assert len(response.json()["items"]) <= 2


def test_page_beyond_last_returns_empty_items(client: TestClient):
    """Requesting a page past the end should return items=[] not 404."""
    response = client.get("/tasks", params={"page": 9999, "size": 10})
    assert response.status_code == 200
    assert response.json()["items"] == []
```

---

### Testing OpenAPI Schema Generation

**FastAPI auto-generates your OpenAPI schema from your route definitions and Pydantic models. Test it — it's your API's public contract, and it can silently drift from your implementation.**

```python
def test_openapi_schema_endpoint_returns_200(client: TestClient):
    """The schema endpoint must always be accessible."""
    response = client.get("/openapi.json")
    assert response.status_code == 200


def test_openapi_schema_has_required_structure(client: TestClient):
    """Schema must follow OpenAPI 3.x structure."""
    response = client.get("/openapi.json")
    schema = response.json()

    assert "openapi" in schema
    assert schema["openapi"].startswith("3.")
    assert "info" in schema
    assert "paths" in schema


def test_openapi_documents_all_endpoints(client: TestClient):
    """Every endpoint must appear in the OpenAPI schema."""
    schema = client.get("/openapi.json").json()
    paths = schema["paths"]

    assert "/tasks" in paths
    assert "/tasks/{task_id}" in paths

    # Check HTTP methods are documented
    assert "post" in paths["/tasks"]    # Create
    assert "get" in paths["/tasks"]     # List


def test_response_model_is_reflected_in_schema(client: TestClient):
    """TaskResponse fields must appear in the schema components."""
    schema = client.get("/openapi.json").json()
    schemas = schema.get("components", {}).get("schemas", {})

    # TaskResponse should be in the schemas
    assert "TaskResponse" in schemas
    properties = schemas["TaskResponse"]["properties"]

    expected_fields = {"id", "title", "priority", "completed"}
    assert expected_fields.issubset(properties.keys())
```

> "If you rename a field in `TaskResponse` from `priority` to `importance`, the last test above fails immediately. You know before your frontend team does. That's the contract being enforced."

---

# PART 4: ISOLATING YOUR CODE

## 4.1 The Shared State Problem

**Let's run multiple tests and see what happens.**

```python
# test_tasks.py — ❌ These tests INTERFERE with each other
from fastapi.testclient import TestClient
from app import app

client = TestClient(app)


def test_first_task_gets_id_1():
    response = client.post("/tasks", json={"title": "First"})
    assert response.json()["id"] == 1  # ✅ Passes


def test_second_task_also_gets_id_1():
    response = client.post("/tasks", json={"title": "Second"})
    assert response.json()["id"] == 1  # ❌ FAILS! id is 2!
```

**Run it:**

```
$ pytest test_tasks.py -v

test_first_task_gets_id_1 PASSED
test_second_task_also_gets_id_1 FAILED
    assert response.json()["id"] == 1
    AssertionError: assert 2 == 1
```

**What happened?**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE SHARED STATE PROBLEM                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  test_first_task:                                               │
│  ── tasks_db is empty, _id_counter is 0                         │
│  ── Creates task → _id_counter becomes 1                        │
│  ── tasks_db now has {1: {...}}                                  │
│  ── ✅ id == 1                                                   │
│                                                                 │
│  test_second_task:                                              │
│  ── tasks_db STILL has {1: {...}} from previous test!           │
│  ── _id_counter is STILL 1                                      │
│  ── Creates task → _id_counter becomes 2                        │
│  ── tasks_db now has {1: {...}, 2: {...}}                        │
│  ── ❌ id == 2, not 1                                            │
│                                                                 │
│  The module-level tasks_db and _id_counter PERSIST between      │
│  tests. State from test 1 LEAKED into test 2.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "Even worse: what happens if you run these tests in a different ORDER? `test_second` runs first? Now `test_first` passes and `test_second` fails. Your tests are ORDER-DEPENDENT. This is a testing nightmare."

**The quick hack (don't do this):**

```python
# ❌ Fragile: reaches into app internals to reset state
from app import tasks_db, _id_counter

@pytest.fixture(autouse=True)
def reset_db():
    tasks_db.clear()
    # Can't even reset _id_counter — it's a module-level int!
    yield
    tasks_db.clear()
```

This is fragile because you need to know every piece of internal state. If the app adds another module-level variable, your reset breaks silently.

**The real solution: dependency injection.**

---

## 4.2 dependency_overrides (FastAPI's Secret Weapon)

**Step 1: Refactor the app to use a dependency for storage.**

> "Connection to Week 3 Lecture 4: you learned `Depends()`. Now you'll see WHY dependency injection matters — not just for code organization, but for TESTABILITY."

```python
# storage.py — Extract storage into its own module
class TaskStorage:
    def __init__(self) -> None:
        self._tasks: dict[int, dict] = {}
        self._next_id: int = 0
    
    def create(self, data: dict) -> dict:
        self._next_id += 1
        data["id"] = self._next_id
        self._tasks[self._next_id] = data
        return data
    
    def get(self, task_id: int) -> dict | None:
        return self._tasks.get(task_id)
    
    def list_all(self) -> list[dict]:
        return list(self._tasks.values())
```

```python
# app.py — Refactored with dependency injection
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, Field
from storage import TaskStorage

app = FastAPI()

# Module-level instance (production)
_storage = TaskStorage()


def get_storage() -> TaskStorage:
    """Dependency: provides the storage instance."""
    return _storage


class TaskCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    priority: int = Field(ge=1, le=5, default=3)


class TaskResponse(BaseModel):
    id: int
    title: str
    priority: int
    completed: bool


@app.post("/tasks", status_code=201)
async def create_task(
    task: TaskCreate,
    storage: TaskStorage = Depends(get_storage),
) -> TaskResponse:
    task_data = {**task.model_dump(), "completed": False}
    created = storage.create(task_data)
    return TaskResponse(**created)


@app.get("/tasks/{task_id}")
async def get_task(
    task_id: int,
    storage: TaskStorage = Depends(get_storage),
) -> TaskResponse:
    task_data = storage.get(task_id)
    if task_data is None:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")
    return TaskResponse(**task_data)
```

**Step 2: Override the dependency in tests.**

```python
# test_tasks.py — ✅ Isolated tests with dependency_overrides
import pytest
from fastapi.testclient import TestClient
from app import app, get_storage
from storage import TaskStorage


@pytest.fixture
def client():
    """Each test gets a FRESH storage instance."""
    test_storage = TaskStorage()   # Brand new, empty storage
    
    # Tell FastAPI: when anything asks for get_storage, 
    # give them test_storage instead
    app.dependency_overrides[get_storage] = lambda: test_storage
    
    yield TestClient(app)
    
    # Clean up: remove the override
    app.dependency_overrides.clear()


def test_first_task_gets_id_1(client: TestClient):
    response = client.post("/tasks", json={"title": "First"})
    assert response.json()["id"] == 1  # ✅ Fresh storage!


def test_second_task_also_gets_id_1(client: TestClient):
    response = client.post("/tasks", json={"title": "Second"})
    assert response.json()["id"] == 1  # ✅ ALSO 1! Fresh storage each time!
```

**Both pass. Every test starts clean.**

```
$ pytest test_tasks.py -v

test_first_task_gets_id_1 PASSED
test_second_task_also_gets_id_1 PASSED
```

**How `dependency_overrides` works:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 dependency_overrides                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  app.dependency_overrides is a DICT:                            │
│  {                                                              │
│    original_function: replacement_function                      │
│  }                                                              │
│                                                                 │
│  When FastAPI resolves Depends(get_storage):                    │
│                                                                 │
│  PRODUCTION (no overrides):                                     │
│  Depends(get_storage)                                           │
│       │                                                         │
│       ▼                                                         │
│  get_storage()  →  returns _storage (module-level, shared)      │
│                                                                 │
│                                                                 │
│  TEST (with override):                                          │
│  app.dependency_overrides[get_storage] = lambda: test_storage   │
│                                                                 │
│  Depends(get_storage)                                           │
│       │                                                         │
│       ▼                                                         │
│  Override found! Call lambda instead                             │
│       │                                                         │
│       ▼                                                         │
│  lambda: test_storage  →  returns test_storage (fresh, per-test)│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In the vending machine analogy:**

> "`dependency_overrides` is like opening the machine's service panel and swapping the product tray. Instead of whatever random stock was left from previous customers, you load a KNOWN set of test products into KNOWN slots. Every test starts from the exact same state. That's isolation."

---

## 4.3 Overriding Yield Dependencies

**Connection to Week 3 Lecture 4:**

> "You learned yield dependencies for setup/teardown patterns. Overriding them works exactly the same way — the override can also be a generator."

```python
# In your app — a yield dependency
def get_audit_logger():
    """Provides an audit logger; flushes on teardown."""
    logger = AuditLogger(destination="production.log")
    try:
        yield logger
    finally:
        logger.flush()

@app.post("/tasks", status_code=201)
async def create_task(
    task: TaskCreate,
    storage: TaskStorage = Depends(get_storage),
    audit: AuditLogger = Depends(get_audit_logger),
) -> TaskResponse:
    created = storage.create({**task.model_dump(), "completed": False})
    audit.log(f"Task created: {created['id']}")
    return TaskResponse(**created)
```

```python
# In your tests — override with a test version
def override_audit_logger():
    """Test version: logs to memory, not to file."""
    logger = FakeAuditLogger()  # Stores logs in a list
    try:
        yield logger
    finally:
        pass  # No flush needed in tests


@pytest.fixture
def client():
    test_storage = TaskStorage()
    app.dependency_overrides[get_storage] = lambda: test_storage
    app.dependency_overrides[get_audit_logger] = override_audit_logger
    
    yield TestClient(app)
    
    app.dependency_overrides.clear()
```

**The override must match the pattern:**

```
┌─────────────────────────────────────────────────────────────────┐
│            MATCHING DEPENDENCY PATTERNS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Original uses return → Override must use return (or lambda)    │
│                                                                 │
│      def get_storage():                                         │
│          return storage                                         │
│                                                                 │
│      override = lambda: test_storage     ✅                     │
│                                                                 │
│                                                                 │
│  Original uses yield → Override should use yield                │
│                                                                 │
│      def get_logger():                                          │
│          logger = Logger()                                      │
│          yield logger                                           │
│          logger.close()                                         │
│                                                                 │
│      def override_logger():                                     │
│          yield FakeLogger()              ✅                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "You'll use yield dependency overrides constantly in Week 6 when you inject database sessions. The pattern: production yields a real session that commits; tests yield a session that rolls back. Same interface, different behavior."

---

## 4.4 Unit vs Integration: When to Override, When Not To

**This is a judgment call, not a rule. But here's the framework.**

```
┌─────────────────────────────────────────────────────────────────┐
│             UNIT vs INTEGRATION TESTS FOR APIs                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  UNIT TEST (override dependencies)                              │
│  ─────────────────────────────────                              │
│  Replace real dependencies with fakes/mocks.                    │
│  Test your endpoint logic IN ISOLATION.                         │
│                                                                 │
│  ├─ Fast (no real I/O)                                          │
│  ├─ Deterministic (no flaky network/database)                   │
│  ├─ Pinpoints failures (if it breaks, it's YOUR code)           │
│  └─ Risk: your fakes might not match real behavior              │
│                                                                 │
│  Example: Override get_storage with an in-memory fake           │
│                                                                 │
│                                                                 │
│  INTEGRATION TEST (use real dependencies)                       │
│  ────────────────────────────────────────                       │
│  Use the actual dependencies (real database, real service).     │
│  Test that everything works TOGETHER.                           │
│                                                                 │
│  ├─ Realistic (catches integration bugs)                        │
│  ├─ Slower (real I/O, setup/teardown)                           │
│  ├─ Can be flaky (network, timing)                              │
│  └─ Harder to pinpoint failures (which layer broke?)            │
│                                                                 │
│  Example: Use a real test database (Week 6)                     │
│                                                                 │
│                                                                 │
│  THE TESTING PYRAMID (from Week 2):                             │
│                                                                 │
│          /\                                                     │
│         /  \        Few end-to-end tests (slow, realistic)      │
│        /    \                                                   │
│       /──────\      Some integration tests (medium)             │
│      /        \                                                 │
│     /──────────\    Many unit tests (fast, isolated)            │
│    /            \                                               │
│                                                                 │
│  For your Week 3-4 project (in-memory storage):                 │
│  Everything is effectively a unit test — there's no external    │
│  system to integrate with yet. That changes in Week 6.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 Overrides vs Mocks (Connection to Week 2)

**Connection to what you've learned:**

> "In Week 2, you learned `unittest.mock` for creating test doubles. How does that compare to `dependency_overrides`? They solve the same problem — isolation — but at different levels."

```
┌─────────────────────────────────────────────────────────────────┐
│          dependency_overrides vs unittest.mock                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  dependency_overrides                                           │
│  ────────────────────                                           │
│  FastAPI-specific. Replaces an entire dependency function.      │
│  Works at the FRAMEWORK level — FastAPI knows about it.         │
│  Clean, explicit, purpose-built for API testing.                │
│                                                                 │
│    app.dependency_overrides[get_storage] = lambda: fake         │
│                                                                 │
│  Best for: replacing dependencies injected via Depends()        │
│                                                                 │
│                                                                 │
│  unittest.mock (Mock, patch)                                    │
│  ───────────────────────────                                    │
│  General-purpose. Replaces any object/function at runtime.      │
│  Works at the PYTHON level — patches module attributes.         │
│  Flexible but can lead to brittle tests.                        │
│                                                                 │
│    with patch("app.some_service") as mock_service:              │
│        mock_service.return_value = fake_result                  │
│                                                                 │
│  Best for: replacing things NOT injected via Depends()          │
│  (external libraries, module-level functions, etc.)             │
│                                                                 │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  If it's a Depends() → use dependency_overrides                 │
│  If it's NOT a Depends() → use unittest.mock                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Example: combining both in one test:**

```python
from unittest.mock import patch

def test_create_task_with_notification(client: TestClient):
    """Test that creating a task sends a notification."""
    
    # dependency_overrides already handle storage (via fixture)
    
    # unittest.mock handles the notification service
    # (not injected via Depends — it's a module-level call)
    with patch("app.send_notification") as mock_notify:
        response = client.post("/tasks", json={"title": "Important"})
        
        assert response.status_code == 201
        mock_notify.assert_called_once()
```

---

## 4.6 Testing Authenticated Endpoints

**Connection to Week 3 Lecture 4:**

> "You learned `Depends()` for injecting dependencies. `get_current_user` is just a dependency — which means `dependency_overrides` is the perfect tool for auth testing. No real tokens. No JWT decode. No database lookup. You inject a known user and test what your endpoint does WITH that user."

**The app code with auth (Week 9 will cover the full implementation):**

```python
# app.py extension — simplified auth for testing demonstration
from pydantic import BaseModel
from fastapi.security import OAuth2PasswordBearer
from fastapi import Depends, HTTPException, status

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")


class CurrentUser(BaseModel):
    id: int
    username: str
    role: str   # "user" | "admin"


async def get_current_user(
    token: str = Depends(oauth2_scheme)
) -> CurrentUser:
    """Decode token and return current user. Full implementation in Week 9."""
    # ... JWT decode, DB lookup ...
    raise NotImplementedError  # Placeholder


async def require_admin(
    current_user: CurrentUser = Depends(get_current_user),
) -> CurrentUser:
    """Secondary dependency: requires admin role."""
    if current_user.role != "admin":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin access required"
        )
    return current_user


@app.get("/me")
async def get_me(current_user: CurrentUser = Depends(get_current_user)):
    return {"id": current_user.id, "username": current_user.username}


@app.delete("/admin/users/{user_id}", status_code=204)
async def delete_user(
    user_id: int,
    _: CurrentUser = Depends(require_admin),   # admin only
):
    ...  # delete logic
```

**Now the test file:**

```python
# tests/test_auth.py
import pytest
from fastapi.testclient import TestClient
from app import app, get_current_user, CurrentUser


# ── Helper ──────────────────────────────────────────────────────
def fake_user(role: str = "user", username: str = "testuser") -> CurrentUser:
    return CurrentUser(id=42, username=username, role=role)


# ── Fixtures ─────────────────────────────────────────────────────
@pytest.fixture
def unauth_client():
    """No auth override — requests arrive without a valid token."""
    # NOTE: do NOT override get_current_user here.
    # OAuth2PasswordBearer raises 401 if the Authorization header is absent.
    yield TestClient(app)


@pytest.fixture
def user_client():
    """Authenticated as a regular user (role='user')."""
    app.dependency_overrides[get_current_user] = lambda: fake_user(role="user")
    yield TestClient(app)
    app.dependency_overrides.clear()


@pytest.fixture
def admin_client():
    """Authenticated as an admin user (role='admin')."""
    app.dependency_overrides[get_current_user] = lambda: fake_user(role="admin")
    yield TestClient(app)
    app.dependency_overrides.clear()


# ── Tests ─────────────────────────────────────────────────────────
def test_no_token_returns_401(unauth_client: TestClient):
    """Unauthenticated request to a protected endpoint → 401."""
    response = unauth_client.get("/me")
    assert response.status_code == 401


def test_authenticated_user_sees_own_profile(user_client: TestClient):
    """Valid user → 200 with correct username."""
    response = user_client.get("/me")
    assert response.status_code == 200
    assert response.json()["username"] == "testuser"


def test_regular_user_cannot_access_admin_endpoint(user_client: TestClient):
    """User with role='user' → 403 Forbidden on admin route."""
    response = user_client.delete("/admin/users/99")
    assert response.status_code == 403
    assert "Admin" in response.json()["detail"]


def test_admin_can_access_admin_endpoint(admin_client: TestClient):
    """User with role='admin' → allowed on admin route."""
    response = admin_client.delete("/admin/users/99")
    assert response.status_code == 204
```

**Testing user-specific data isolation:**

```python
def test_user_can_only_see_own_tasks(client: TestClient, storage: TaskStorage):
    """User A should not see User B's tasks."""
    # Simulate User A creating a task
    app.dependency_overrides[get_current_user] = lambda: fake_user(id=1, username="alice")
    with TestClient(app) as alice_client:
        alice_client.post("/tasks", json={"title": "Alice's secret task"})

    # Simulate User B listing tasks
    app.dependency_overrides[get_current_user] = lambda: fake_user(id=2, username="bob")
    with TestClient(app) as bob_client:
        response = bob_client.get("/tasks")
        task_titles = [t["title"] for t in response.json()]
        assert "Alice's secret task" not in task_titles

    app.dependency_overrides.clear()
```

```
┌─────────────────────────────────────────────────────────────────┐
│           AUTH TESTING — WHAT YOU'RE ACTUALLY TESTING           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NOT testing: JWT signature validity                            │
│  NOT testing: Password hashing correctness                      │
│  NOT testing: Token expiry                                      │
│  (Those are unit tests on your auth utility functions)          │
│                                                                 │
│  YOU ARE testing:                                               │
│  ✅ That unauthenticated requests are rejected (401)            │
│  ✅ That unauthorized roles are rejected (403)                  │
│  ✅ That authorized roles can proceed (200/204)                 │
│  ✅ That user-specific data isolation works correctly           │
│  ✅ That the dependency chain (get_current_user →               │
│       require_admin) resolves correctly                         │
│                                                                 │
│  This is the HTTP contract for authentication.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You have an endpoint PATCH /tasks/{id} that requires the       │
│  current user to be the OWNER of the task (user.id must match   │
│  task.owner_id), otherwise return 403. Write TWO fixtures and   │
│  TWO tests: one showing the owner can update, and one showing   │
│  a different user cannot.                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION

@pytest.fixture
def owner_client():
    app.dependency_overrides[get_current_user] = lambda: fake_user(id=1)
    yield TestClient(app)
    app.dependency_overrides.clear()


@pytest.fixture
def other_client():
    app.dependency_overrides[get_current_user] = lambda: fake_user(id=2)
    yield TestClient(app)
    app.dependency_overrides.clear()


def test_owner_can_update_task(owner_client, storage):
    # Create a task owned by user id=1
    task = storage.create({"title": "My task", "priority": 3,
                           "completed": False, "owner_id": 1})
    response = owner_client.patch(
        f"/tasks/{task['id']}",
        json={"title": "Updated title"}
    )
    assert response.status_code == 200


def test_non_owner_cannot_update_task(other_client, storage):
    task = storage.create({"title": "Not yours", "priority": 3,
                           "completed": False, "owner_id": 1})
    response = other_client.patch(
        f"/tasks/{task['id']}",
        json={"title": "Hacked title"}
    )
    assert response.status_code == 403
```

---

## 4.7 Testing Middleware

**The pipeline diagram from Part 1 shows middleware as the outermost layer. You cannot test middleware by calling functions directly — you MUST go through the HTTP pipeline. TestClient is the only way.**

**App code (middleware extension):**

```python
# app.py extension
import time
from starlette.middleware.base import BaseHTTPMiddleware
from fastapi import Request

class RequestIDMiddleware(BaseHTTPMiddleware):
    """Echoes X-Request-ID from request to response; generates one if absent."""
    async def dispatch(self, request: Request, call_next):
        request_id = request.headers.get("X-Request-ID", "auto-generated")
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response


class TimingMiddleware(BaseHTTPMiddleware):
    """Adds X-Process-Time (ms) to every response."""
    async def dispatch(self, request: Request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        elapsed_ms = (time.perf_counter() - start) * 1000
        response.headers["X-Process-Time"] = f"{elapsed_ms:.2f}ms"
        return response


app.add_middleware(TimingMiddleware)
app.add_middleware(RequestIDMiddleware)
```

**Tests:**

```python
# tests/test_middleware.py

def test_request_id_echoed_when_provided(client: TestClient):
    """Provided X-Request-ID should appear verbatim in response headers."""
    response = client.get("/tasks", headers={"X-Request-ID": "trace-abc-123"})
    assert response.headers.get("x-request-id") == "trace-abc-123"


def test_request_id_generated_when_absent(client: TestClient):
    """Missing X-Request-ID → middleware adds a generated value."""
    response = client.get("/tasks")
    assert "x-request-id" in response.headers
    assert response.headers["x-request-id"] == "auto-generated"


def test_timing_header_present_on_all_responses(client: TestClient):
    """Every response — success and error — should have X-Process-Time."""
    for path in ["/tasks", "/tasks/99999", "/tasks/abc"]:
        response = client.get(path)
        assert "x-process-time" in response.headers, (
            f"X-Process-Time missing on GET {path} → {response.status_code}"
        )


def test_timing_header_is_valid_number(client: TestClient):
    """X-Process-Time must parse as a non-negative float."""
    response = client.get("/tasks")
    raw = response.headers["x-process-time"]
    # Strip unit suffix if present ("12.34ms" → 12.34)
    value = float(raw.rstrip("ms"))
    assert value >= 0


def test_middleware_runs_on_error_responses(client: TestClient):
    """Middleware must wrap error responses too — not just successes."""
    response = client.get("/tasks/99999")
    assert response.status_code == 404
    # Middleware should have run even on this error response
    assert "x-request-id" in response.headers
    assert "x-process-time" in response.headers
```

> "The last test is the most important. A common middleware bug: the middleware calls `call_next()` but only modifies the response on the HAPPY path. If an exception short-circuits `call_next`, the middleware's response modification never runs. TestClient lets you catch this."

---

## 4.8 Mocking External HTTP Calls

**Real APIs call other APIs. You cannot let those calls happen during tests — the network is slow, flaky, costs money, and makes tests non-deterministic. You must mock the external HTTP layer.**

**Two tools: `respx` (httpx-specific, purpose-built) and `unittest.mock` (general-purpose).**

**App code (external API integration):**

```python
# app.py extension
import httpx

WEATHER_API_BASE = "https://api.openweathermap.org/data/2.5/weather"
WEATHER_API_KEY = "your-api-key"

@app.get("/weather/{city}")
async def get_weather(city: str) -> dict:
    """Fetch weather from OpenWeatherMap and return simplified data."""
    async with httpx.AsyncClient() as http:
        resp = await http.get(
            WEATHER_API_BASE,
            params={"q": city, "appid": WEATHER_API_KEY, "units": "metric"}
        )
    if resp.status_code == 404:
        raise HTTPException(
            status_code=404,
            detail=f"City '{city}' not found"
        )
    if resp.status_code != 200:
        raise HTTPException(
            status_code=502,
            detail="Weather service unavailable"
        )
    data = resp.json()
    return {
        "city": city,
        "temperature": data["main"]["temp"],
        "description": data["weather"][0]["description"],
    }
```

---

### Using respx

**Setup:**

```toml
# pyproject.toml
[project.optional-dependencies]
test = ["pytest", "pytest-asyncio", "httpx", "respx"]
```

**Tests:**

```python
# tests/test_weather.py
import respx
import httpx

WEATHER_API_BASE = "https://api.openweathermap.org/data/2.5/weather"

FAKE_WEATHER_RESPONSE = {
    "main": {"temp": 18.5},
    "weather": [{"description": "partly cloudy"}]
}


@respx.mock
def test_get_weather_success(client: TestClient):
    """Happy path: external API responds → our endpoint returns shaped data."""
    respx.get(WEATHER_API_BASE).mock(
        return_value=httpx.Response(200, json=FAKE_WEATHER_RESPONSE)
    )

    response = client.get("/weather/London")

    assert response.status_code == 200
    data = response.json()
    assert data["city"] == "London"
    assert data["temperature"] == 18.5
    assert data["description"] == "partly cloudy"


@respx.mock
def test_unknown_city_returns_404(client: TestClient):
    """When external API returns 404 for city, our endpoint should too."""
    respx.get(WEATHER_API_BASE).mock(
        return_value=httpx.Response(404, json={"message": "city not found"})
    )

    response = client.get("/weather/MadeUpCity")

    assert response.status_code == 404
    assert "MadeUpCity" in response.json()["detail"]


@respx.mock
def test_weather_service_down_returns_502(client: TestClient):
    """When external API fails (500), our endpoint should return 502."""
    respx.get(WEATHER_API_BASE).mock(
        return_value=httpx.Response(500, json={"message": "internal error"})
    )

    response = client.get("/weather/London")

    assert response.status_code == 502
    assert "unavailable" in response.json()["detail"].lower()


@respx.mock
def test_correct_api_params_are_sent(client: TestClient):
    """Verify our endpoint sends the correct params to the external API."""
    mock_weather = respx.get(WEATHER_API_BASE).mock(
        return_value=httpx.Response(200, json=FAKE_WEATHER_RESPONSE)
    )

    client.get("/weather/Paris")

    assert mock_weather.called
    request_url = str(mock_weather.calls.last.request.url)
    assert "Paris" in request_url
    assert "metric" in request_url    # units=metric
```

---

### Using unittest.mock as an alternative

```python
# When respx is not available or you need to mock at the object level
from unittest.mock import patch, AsyncMock, MagicMock

def test_weather_with_unittest_mock(client: TestClient):
    """Same test using unittest.mock instead of respx."""
    mock_response = MagicMock()
    mock_response.status_code = 200
    mock_response.json.return_value = FAKE_WEATHER_RESPONSE

    # Patch the AsyncClient context manager
    mock_http = AsyncMock()
    mock_http.get = AsyncMock(return_value=mock_response)

    with patch("app.httpx.AsyncClient") as mock_client_cls:
        mock_client_cls.return_value.__aenter__ = AsyncMock(return_value=mock_http)
        mock_client_cls.return_value.__aexit__ = AsyncMock(return_value=False)

        response = client.get("/weather/London")

    assert response.status_code == 200
    assert response.json()["temperature"] == 18.5
```

```
┌─────────────────────────────────────────────────────────────────┐
│              respx vs unittest.mock                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  respx                                                          │
│  ── Intercepts at the httpx transport level                     │
│  ── URL/method matching is first-class                          │
│  ── Inspect actual requests (url, headers, body)                │
│  ── Clean decorator or context manager API                      │
│  ── ✅ Use when your app uses httpx                             │
│                                                                 │
│  unittest.mock                                                  │
│  ── Patches Python objects at module level                      │
│  ── Verbose for async context managers (nested AsyncMock)       │
│  ── Works with any HTTP library (requests, aiohttp, httpx)      │
│  ── ✅ Use when respx isn't available or you mock non-httpx code │
│                                                                 │
│  RULE: Your app uses httpx → use respx. It's purpose-built.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your endpoint GET /exchange-rate/{currency} calls              │
│  https://api.exchangerate.host/latest?base={currency}.          │
│  Using respx, write a test that: (1) mocks a successful         │
│  response returning a rate of 1.25, and (2) verifies the        │
│  mock was called with the correct currency in the URL.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION
import respx
import httpx

EXCHANGE_API = "https://api.exchangerate.host/latest"

@respx.mock
def test_exchange_rate_success(client: TestClient):
    mock_api = respx.get(EXCHANGE_API).mock(
        return_value=httpx.Response(
            200,
            json={"base": "USD", "rates": {"EUR": 1.25}}
        )
    )

    response = client.get("/exchange-rate/USD")

    assert response.status_code == 200
    assert response.json()["rate"] == 1.25

    # Verify correct params were sent
    assert mock_api.called
    request_url = str(mock_api.calls.last.request.url)
    assert "USD" in request_url
```

---

# PART 5: ASYNC TESTING & TEST ARCHITECTURE

## 5.1 When TestClient Isn't Enough

**TestClient handles 90% of API testing. But it's synchronous under the hood.**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHEN TO MOVE BEYOND TestClient                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TestClient is FINE when:                                       │
│  ├─ Testing request/response cycles (GET, POST, etc.)           │
│  ├─ Testing validation, status codes, response bodies           │
│  ├─ Testing dependency overrides                                │
│  └─ ✅ This covers MOST of your Week 3-4 project tests          │
│                                                                 │
│  You NEED async testing when:                                   │
│  ├─ Testing WebSocket endpoints (Week 12)                       │
│  ├─ Testing Server-Sent Events (SSE)                            │
│  ├─ Your fixtures are async (async database sessions — Week 6)  │
│  ├─ You want to test concurrent request behavior                │
│  └─ Your test itself needs to do async I/O                      │
│                                                                 │
│  DON'T use async tests just because your endpoints are          │
│  async def. TestClient handles that transparently.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 httpx.AsyncClient with ASGITransport

**When you do need async tests, `httpx.AsyncClient` is the tool.**

**Setup in `pyproject.toml`:**

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"

[project.optional-dependencies]
test = [
    "pytest",
    "pytest-asyncio",
    "httpx",
]
```

### pytest-asyncio Modes

**Paying off tech debt from Week 2, Lecture 2: `asyncio_mode = "auto"` has been in your pyproject.toml. Here's exactly what it controls.**

```
┌─────────────────────────────────────────────────────────────────┐
│              pytest-asyncio MODES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MODE: "auto"                                                   │
│  ── Any async def test_* runs as an asyncio test automatically  │
│  ── No extra decorator required                                 │
│  ── Lowest friction                                             │
│  ── ✅ Use this — it's the right default for async-heavy APIs   │
│                                                                 │
│  MODE: "strict"                                                 │
│  ── Only test functions decorated with @pytest.mark.asyncio     │
│     are treated as async tests                                  │
│  ── All others are treated as synchronous (even if async def)   │
│  ── Requires more ceremony but is explicit                      │
│  ── ✅ Use in large teams needing fine-grained control          │
│                                                                 │
│  MODE: "legacy"  (before pytest-asyncio 0.21)                   │
│  ── Old auto-detection heuristic                                │
│  ── Deprecated — do not use in new projects                     │
│  ── ⚠️ You will encounter this in old codebases                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In code, the difference:**

```python
# ── asyncio_mode = "auto" (your pyproject.toml is already set this way) ──
# No decorator needed. Write async def and it just works.

async def test_create_task(async_client: AsyncClient):
    response = await async_client.post("/tasks", json={"title": "Auto mode"})
    assert response.status_code == 201


# ── asyncio_mode = "strict" (if you switch to strict later) ──
# Must explicitly mark EVERY async test function.

@pytest.mark.asyncio
async def test_create_task(async_client: AsyncClient):
    response = await async_client.post("/tasks", json={"title": "Strict mode"})
    assert response.status_code == 201

# In strict mode, an unmarked async def test is collected but
# pytest detects it never actually awaited — and raises a warning.
```

---

**Your first async API test:**

```python
# test_tasks_async.py
import pytest
from httpx import AsyncClient, ASGITransport
from app import app


@pytest.fixture
async def async_client():
    """Async test client — truly async, no sync wrapper."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client


async def test_create_task(async_client: AsyncClient):
    response = await async_client.post(
        "/tasks",
        json={"title": "Async test", "priority": 2}
    )
    
    assert response.status_code == 201
    assert response.json()["title"] == "Async test"
```

**Compare the two approaches:**

```
┌─────────────────────────────────────────────────────────────────┐
│            TestClient vs AsyncClient                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TestClient (sync):                                             │
│  ──────────────────                                             │
│     def test_create(client: TestClient):                        │
│         response = client.post("/tasks", json={...})            │
│         assert response.status_code == 201                      │
│                                                                 │
│  ├─ Plain def test function                                     │
│  ├─ No await needed                                             │
│  ├─ Simpler to write                                            │
│  └─ Uses requests-like interface                                │
│                                                                 │
│                                                                 │
│  AsyncClient (async):                                           │
│  ────────────────────                                           │
│     async def test_create(async_client: AsyncClient):           │
│         response = await async_client.post("/tasks", json={..}) │
│         assert response.status_code == 201                      │
│                                                                 │
│  ├─ async def test function                                     │
│  ├─ Must await every call                                       │
│  ├─ More ceremony, but async-native                             │
│  └─ Uses httpx interface (you'll learn httpx in Week 8)         │
│                                                                 │
│                                                                 │
│  RECOMMENDATION for your project:                               │
│  Start with TestClient. Switch to AsyncClient when              │
│  you have a concrete reason (async fixtures, WebSockets).       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Combining async client with dependency overrides:**

```python
import pytest
from httpx import AsyncClient, ASGITransport
from app import app, get_storage
from storage import TaskStorage


@pytest.fixture
async def async_client():
    test_storage = TaskStorage()
    app.dependency_overrides[get_storage] = lambda: test_storage
    
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client
    
    app.dependency_overrides.clear()


async def test_create_and_retrieve(async_client: AsyncClient):
    # Create
    create_resp = await async_client.post(
        "/tasks", json={"title": "Async roundtrip"}
    )
    task_id = create_resp.json()["id"]
    
    # Retrieve
    get_resp = await async_client.get(f"/tasks/{task_id}")
    assert get_resp.status_code == 200
    assert get_resp.json()["title"] == "Async roundtrip"
```

---

### Concurrent Request Testing

**TestClient is synchronous — you cannot send two requests "at the same time" with it. AsyncClient is truly async, which means `asyncio.gather()` works the way you'd expect.**

```python
# tests/test_concurrent.py
import asyncio
import pytest
from httpx import AsyncClient, ASGITransport
from app import app, get_storage
from storage import TaskStorage


@pytest.fixture
async def async_client():
    test_storage = TaskStorage()
    app.dependency_overrides[get_storage] = lambda: test_storage
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client
    app.dependency_overrides.clear()


async def test_concurrent_task_creation_assigns_unique_ids(
    async_client: AsyncClient,
):
    """10 concurrent POSTs must each get a unique ID — no race conditions."""
    coroutines = [
        async_client.post("/tasks", json={"title": f"Task {i}"})
        for i in range(10)
    ]
    responses = await asyncio.gather(*coroutines)

    # All requests succeeded
    for resp in responses:
        assert resp.status_code == 201, f"Unexpected status: {resp.status_code}"

    # All IDs are unique — no ID assigned twice
    ids = [r.json()["id"] for r in responses]
    assert len(set(ids)) == 10, f"Duplicate IDs detected: {ids}"


async def test_concurrent_reads_return_consistent_data(
    async_client: AsyncClient,
):
    """50 simultaneous reads of the same resource should all agree."""
    create_resp = await async_client.post(
        "/tasks", json={"title": "Shared Task"}
    )
    task_id = create_resp.json()["id"]

    reads = [async_client.get(f"/tasks/{task_id}") for _ in range(50)]
    responses = await asyncio.gather(*reads)

    for resp in responses:
        assert resp.status_code == 200
        assert resp.json()["id"] == task_id
        assert resp.json()["title"] == "Shared Task"
```

```
┌─────────────────────────────────────────────────────────────────┐
│              WHEN TO WRITE CONCURRENT TESTS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE concurrent tests when:                                     │
│  ├─ Testing ID generation or sequence counters                  │
│  ├─ Verifying mutable shared state under concurrent load        │
│  ├─ Checking that rate limiters fire at the correct count       │
│  └─ Testing that connection pool doesn't exhaust under load     │
│                                                                 │
│  DON'T use concurrent tests for:                                │
│  ├─ Validation errors — no concurrency needed                   │
│  ├─ Auth behavior — sequential tests are clearer                │
│  └─ Most standard CRUD operations                               │
│                                                                 │
│  IMPORTANT: Use AsyncClient for concurrent tests.               │
│  TestClient is synchronous — asyncio.gather() with TestClient   │
│  does not give you true concurrency.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Without looking above: write an async test that fires 5        │
│  concurrent DELETE requests to /tasks/1 and asserts that        │
│  exactly ONE returns 204 and the remaining FOUR return 404.     │
│  (This tests that your delete is truly idempotent under         │
│  concurrent load and doesn't double-delete or crash.)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION

async def test_concurrent_deletes_are_idempotent(async_client: AsyncClient):
    # Arrange: create one task
    create_resp = await async_client.post("/tasks", json={"title": "Delete race"})
    task_id = create_resp.json()["id"]

    # Act: 5 concurrent deletes
    deletes = [async_client.delete(f"/tasks/{task_id}") for _ in range(5)]
    responses = await asyncio.gather(*deletes)

    status_codes = [r.status_code for r in responses]

    # Assert: exactly one 204, four 404s
    assert status_codes.count(204) == 1, f"Expected 1×204, got: {status_codes}"
    assert status_codes.count(404) == 4, f"Expected 4×404, got: {status_codes}"
```

---

## 5.3 Test File Organization

**As your test suite grows, structure matters.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  TEST FILE STRUCTURE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  project/                                                       │
│  ├── src/                                                       │
│  │   ├── app.py                                                │
│  │   ├── storage.py                                            │
│  │   ├── models.py                                             │
│  │   └── ...                                                   │
│  │                                                              │
│  └── tests/                                                    │
│      ├── conftest.py          ← Shared fixtures (client, etc.) │
│      ├── factories.py         ← Test data generators           │
│      ├── test_tasks.py        ← Tests for /tasks endpoints     │
│      ├── test_categories.py   ← Tests for /categories          │
│      └── test_tags.py         ← Tests for /tags                │
│                                                                 │
│  RULE: one test file per RESOURCE (endpoint group).             │
│  RULE: shared fixtures go in conftest.py (Week 2 callback).    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**conftest.py — the shared fixture hub:**

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from app import app, get_storage
from storage import TaskStorage


@pytest.fixture
def storage() -> TaskStorage:
    """Fresh storage for each test."""
    return TaskStorage()


@pytest.fixture
def client(storage: TaskStorage) -> TestClient:
    """TestClient with isolated storage."""
    app.dependency_overrides[get_storage] = lambda: storage
    yield TestClient(app)
    app.dependency_overrides.clear()
```

**Now every test file gets `client` for free:**

```python
# tests/test_tasks.py — no fixture definitions needed!
from fastapi.testclient import TestClient


def test_create_task(client: TestClient):
    response = client.post("/tasks", json={"title": "Test"})
    assert response.status_code == 201


def test_list_tasks_empty(client: TestClient):
    response = client.get("/tasks")
    assert response.status_code == 200
    assert response.json() == []
```

> "Remember from Week 2: pytest automatically discovers `conftest.py` files and makes their fixtures available to all test files in the same directory and subdirectories. One fixture definition, used everywhere."

---

## 5.4 Factory Pattern for Test Data

**You'll write `json={"title": "Test", "priority": 3}` hundreds of times. Don't.**

```python
# tests/factories.py
from typing import Any


def make_task_payload(**overrides: Any) -> dict:
    """Build a valid task creation payload with sensible defaults.
    
    Override any field:
        make_task_payload(title="Custom", priority=5)
    """
    defaults = {
        "title": "Test Task",
        "priority": 3,
    }
    return {**defaults, **overrides}
```

**Usage in tests:**

```python
# tests/test_tasks.py
from factories import make_task_payload


def test_create_task(client):
    response = client.post("/tasks", json=make_task_payload())
    assert response.status_code == 201


def test_create_high_priority_task(client):
    payload = make_task_payload(title="Urgent bug", priority=5)
    response = client.post("/tasks", json=payload)
    assert response.json()["priority"] == 5


def test_create_task_with_long_title(client):
    payload = make_task_payload(title="a" * 100)  # Exactly at limit
    response = client.post("/tasks", json=payload)
    assert response.status_code == 201
```

**Why factories matter:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHY FACTORIES?                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT FACTORY:                                               │
│  ── Every test repeats the same boilerplate payload             │
│  ── Adding a required field to TaskCreate?                      │
│     Update 47 test files. Miss one? Test fails.                 │
│                                                                 │
│  WITH FACTORY:                                                  │
│  ── Tests only specify the fields THEY CARE ABOUT              │
│  ── Adding a required field? Update ONE factory.                │
│  ── Tests read clearly: "this test is about priority=5"         │
│     (the title doesn't matter, so the factory picks one)        │
│                                                                 │
│  Factories make tests:                                          │
│  ├─ DRY (Don't Repeat Yourself)                                 │
│  ├─ Resilient to model changes                                  │
│  └─ Focused on what matters (the thing being tested)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Helper fixture for pre-populated storage:**

```python
# tests/conftest.py — add a fixture that creates tasks for you

@pytest.fixture
def populated_client(client: TestClient) -> TestClient:
    """Client with 3 pre-existing tasks."""
    for i in range(1, 4):
        client.post("/tasks", json=make_task_payload(title=f"Task {i}"))
    return client
```

```python
# tests/test_tasks.py
def test_list_returns_all_tasks(populated_client):
    response = populated_client.get("/tasks")
    assert len(response.json()) == 3
```

---

## 5.5 Your Testing Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│             TESTING STRATEGY FOR YOUR PROJECT                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FOR EACH ENDPOINT, TEST:                                       │
│                                                                 │
│  1. HAPPY PATH                                                  │
│     ── Valid input → correct status code + correct body         │
│     ── Default values work                                      │
│     ── Response has expected shape                              │
│                                                                 │
│  2. VALIDATION ERRORS                                           │
│     ── Missing required fields → 422                            │
│     ── Fields violating constraints → 422                       │
│     ── Wrong types → 422                                        │
│     ── Boundary values (min, max, exactly-at-limit)             │
│                                                                 │
│  3. BUSINESS ERRORS                                             │
│     ── Resource not found → 404                                 │
│     ── Duplicate creation → 409 (if applicable)                 │
│     ── Invalid state transition → 400 (if applicable)           │
│                                                                 │
│  4. EDGE CASES                                                  │
│     ── Empty collections → returns []                           │
│     ── Special characters in input                              │
│     ── Very large valid input                                   │
│                                                                 │
│  USE:                                                           │
│  ── TestClient for all request/response tests (Parts 2-4)      │
│  ── dependency_overrides for isolation (Part 4)                 │
│  ── @pytest.mark.parametrize for validation coverage (Part 3)   │
│  ── Factories for test data (Part 5)                            │
│  ── conftest.py for shared fixtures (Part 5)                    │
│                                                                 │
│  TARGET: 80%+ code coverage for your Week 3-4 project.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.6 Testing Lifespan Events

**The `with TestClient(app)` context manager was mentioned in 2.5 as a way to trigger lifespan. Here is what that actually means and how to test it.**

**Modern FastAPI app with a lifespan hook:**

```python
# app.py extension
from contextlib import asynccontextmanager
from fastapi import FastAPI

# A resource that must be initialized before the app handles requests
class ConnectionPool:
    def __init__(self):
        self.ready = False

    async def connect(self):
        # Simulate opening connections
        self.ready = True

    async def disconnect(self):
        self.ready = False


connection_pool: ConnectionPool | None = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ── STARTUP ───────────────────────────────────────────────
    global connection_pool
    connection_pool = ConnectionPool()
    await connection_pool.connect()
    app.state.pool = connection_pool     # store on app.state

    yield                                # App is running and handling requests

    # ── SHUTDOWN ──────────────────────────────────────────────
    await connection_pool.disconnect()
    app.state.pool = None


app = FastAPI(lifespan=lifespan)


@app.get("/health")
async def health_check(request: Request) -> dict:
    pool = request.app.state.pool
    return {"status": "ok", "pool_ready": pool is not None and pool.ready}
```

**Tests:**

```python
# tests/test_lifespan.py
from fastapi.testclient import TestClient
from app import app


def test_startup_runs_when_context_manager_used():
    """With context manager form: startup must have run before first request."""
    with TestClient(app) as client:
        response = client.get("/health")
        assert response.status_code == 200
        assert response.json()["pool_ready"] is True


def test_startup_does_not_run_without_context_manager():
    """Without context manager: startup has NOT run → pool is None."""
    client = TestClient(app)   # No 'with' — no lifespan
    response = client.get("/health")
    # pool was never initialized → pool_ready is False
    assert response.json()["pool_ready"] is False


def test_shutdown_cleans_up_after_context_manager_exits():
    """After the with block, shutdown should have run."""
    with TestClient(app) as client:
        pass  # Do nothing — just trigger lifecycle

    # After exiting the with block, pool should be disconnected
    # (Test this via app.state if your endpoint exposes it)
    assert app.state.pool is None


def test_app_starts_without_raising():
    """The simplest lifespan test: startup should not throw."""
    try:
        with TestClient(app) as client:
            response = client.get("/health")
            assert response.status_code == 200
    except Exception as exc:
        pytest.fail(f"App startup raised an exception: {exc}")
```

```
┌─────────────────────────────────────────────────────────────────┐
│            THE LIFESPAN RULE — MEMORISE THIS                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TestClient(app)                                                │
│  ── Does NOT trigger startup/shutdown                           │
│  ── app.state is uninitialised                                  │
│  ── Fine for: endpoints with no startup dependencies            │
│                                                                 │
│  with TestClient(app) as client:                                │
│  ── startup() runs on __enter__                                 │
│  ── app.state is populated                                      │
│  ── shutdown() runs on __exit__                                 │
│  ── Required for: any endpoint that reads from app.state,       │
│       uses a DB pool, Redis client, or any resource             │
│       initialised in lifespan                                   │
│                                                                 │
│  When in doubt: use the context manager form.                   │
│  The only cost is a few milliseconds of startup time.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.7 Test Coverage Measurement

**Your project requires 80%+ test coverage. Here is how to install, run, and interpret the tool — and what the number actually means.**

**Setup:**

```toml
# pyproject.toml
[project.optional-dependencies]
test = [
    "pytest",
    "pytest-asyncio",
    "httpx",
    "respx",
    "pytest-cov",      # ← add this
]

[tool.coverage.run]
source = ["app", "storage", "models"]   # modules to measure
omit = [
    "tests/*",
    "*/migrations/*",
]

[tool.coverage.report]
show_missing = true
fail_under = 80
```

**Running coverage:**

```bash
# Terminal report — shows which lines are NOT covered
pytest --cov=app --cov-report=term-missing

# HTML report — click through in a browser (most useful)
pytest --cov=app --cov-report=html
# → open htmlcov/index.html

# Both at once
pytest --cov=app --cov-report=term-missing --cov-report=html

# Fail the CI run if coverage drops below 80%
pytest --cov=app --cov-fail-under=80
```

**Reading the output:**

```
---------- coverage: platform darwin, python 3.12 ----------
Name        Stmts   Miss  Cover  Missing
------------------------------------------
app.py         54     10    81%  23, 47-50, 68-72
storage.py     18      0   100%
models.py      12      2    83%  31, 32
------------------------------------------
TOTAL          84     12    86%
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 COVERAGE REPORT ANATOMY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  app.py      54    10    81%   23, 47-50, 68-72                 │
│   │           │     │     │    │                                │
│   │           │     │     │    └── Line numbers NEVER executed  │
│   │           │     │     └─────── % of lines that WERE run     │
│   │           │     └───────────── Lines not covered (count)    │
│   │           └─────────────────── Total executable lines       │
│   └─────────────────────────────── Module name                  │
│                                                                 │
│  "Missing: 47-50" means lines 47, 48, 49, 50 are a block       │
│  that no test executed. Open app.py and look at those lines.   │
│  They are likely inside an error handler or edge-case branch.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to act on the missing lines report:**

```python
# Suppose line 47 is this:
@app.exception_handler(TaskNotFoundError)
async def task_not_found_handler(request, exc):
    return JSONResponse(           # ← line 47: never executed
        status_code=404,
        content={"error": "not_found", "message": str(exc)}
    )

# The fix: write a test that TRIGGERS this handler
def test_custom_not_found_handler_is_invoked(client: TestClient):
    response = client.get("/tasks/99999")
    assert response.status_code == 404
    assert response.json()["error"] == "not_found"   # custom field
    # Now line 47 is covered
```

```
┌─────────────────────────────────────────────────────────────────┐
│         WHAT COVERAGE MEASURES — AND WHAT IT DOESN'T           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ MEASURES:                                                   │
│  ── Whether each line was executed at least once                │
│  ── Which branches of if/else were taken                        │
│     (with --cov-branch for branch coverage)                     │
│  ── Which functions were called                                 │
│                                                                 │
│  ❌ DOES NOT MEASURE:                                           │
│  ── Whether your assertions are CORRECT                         │
│  ── Whether edge cases are MEANINGFUL                           │
│  ── Whether you tested the right scenarios                      │
│                                                                 │
│  A test that calls every line but asserts nothing               │
│  scores 100% coverage and tests nothing.                        │
│                                                                 │
│  80% is a FLOOR, not a GOAL. Use missing-line reports           │
│  as a MAP to find untested code paths — then write              │
│  tests that assert real behavior on those paths.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You run pytest --cov=app --cov-report=term-missing and see:    │
│  app.py    54    8    85%    12, 38-41, 57                      │
│  Line 12 is the first line of a custom exception handler for    │
│  a DuplicateTaskError your app raises on POST /tasks when the   │
│  title already exists. Lines 38-41 are inside an               │
│  `if search and len(search) > 50` guard. Line 57 is a          │
│  `return []` inside a `try/except` block. Write the test        │
│  function names (you don't need to implement them) that would   │
│  cover each group of missing lines.                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION — test names that map to each missing block

# Line 12 — DuplicateTaskError handler
def test_duplicate_title_returns_409_with_custom_error_format():
    # POST /tasks with same title twice → triggers handler on line 12
    ...

# Lines 38-41 — search > 50 chars guard
def test_search_query_over_50_characters_is_handled():
    # GET /tasks?search=<51-char string> → enters the guard
    ...

# Line 57 — return [] inside except
def test_list_tasks_returns_empty_list_when_storage_raises():
    # Force storage.list_all() to raise → hits the except branch
    ...
```

---

## 5.8 Testing Background Tasks

**FastAPI's `BackgroundTasks` runs functions AFTER the response is sent to the client. Testing them requires understanding one critical fact about TestClient.**

**App code (background tasks):**

```python
# app.py extension
from fastapi import BackgroundTasks
import logging

logger = logging.getLogger(__name__)


def send_task_created_email(task_id: int, owner_email: str) -> None:
    """Send an email notification. Production: call email service here."""
    logger.info(f"Sending email to {owner_email} for task {task_id}")


def update_analytics_counter(task_id: int) -> None:
    """Increment analytics. Production: write to analytics DB."""
    logger.info(f"Analytics updated for task {task_id}")


@app.post("/tasks", status_code=201)
async def create_task(
    task: TaskCreate,
    background_tasks: BackgroundTasks,
    storage: TaskStorage = Depends(get_storage),
) -> TaskResponse:
    task_data = {**task.model_dump(), "completed": False}
    created = storage.create(task_data)

    # Schedule background work — runs after this response is sent
    background_tasks.add_task(
        send_task_created_email,
        task_id=created["id"],
        owner_email="owner@example.com"
    )
    background_tasks.add_task(update_analytics_counter, task_id=created["id"])

    return TaskResponse(**created)
```

**Tests:**

```python
# tests/test_background_tasks.py
from unittest.mock import patch, call


def test_creating_task_schedules_email(client: TestClient):
    """Creating a task should trigger the email notification function."""
    with patch("app.send_task_created_email") as mock_email:
        response = client.post("/tasks", json={"title": "Email me"})

        assert response.status_code == 201
        task_id = response.json()["id"]

        # TestClient executes background tasks synchronously —
        # by the time client.post() returns, the tasks have already run.
        mock_email.assert_called_once_with(
            task_id=task_id,
            owner_email="owner@example.com"
        )


def test_creating_task_schedules_analytics(client: TestClient):
    """Creating a task should trigger the analytics update."""
    with patch("app.update_analytics_counter") as mock_analytics:
        response = client.post("/tasks", json={"title": "Track me"})

        assert response.status_code == 201
        mock_analytics.assert_called_once_with(task_id=response.json()["id"])


def test_both_background_tasks_are_scheduled(client: TestClient):
    """Both background tasks must be scheduled on every create."""
    with (
        patch("app.send_task_created_email") as mock_email,
        patch("app.update_analytics_counter") as mock_analytics,
    ):
        client.post("/tasks", json={"title": "Both tasks"})

        mock_email.assert_called_once()
        mock_analytics.assert_called_once()


def test_background_task_failure_does_not_affect_http_response(client: TestClient):
    """A crashing background task should NOT cause a 500 response.
    
    This test DOCUMENTS the behavior. If your app lets background
    task errors propagate as HTTP errors, this test will catch that
    regression the moment you change the behavior.
    """
    with patch("app.send_task_created_email", side_effect=Exception("SMTP down")):
        response = client.post("/tasks", json={"title": "Failing email"})
        # The HTTP response was already sent before the task ran.
        # TestClient may raise or not depending on FastAPI's error handling.
        # Document whichever behavior your app actually has:
        assert response.status_code == 201


def test_no_background_tasks_on_failed_creation(client: TestClient):
    """If the endpoint fails (422), no background tasks should be scheduled."""
    with (
        patch("app.send_task_created_email") as mock_email,
        patch("app.update_analytics_counter") as mock_analytics,
    ):
        response = client.post("/tasks", json={"title": ""})  # Invalid: empty

        assert response.status_code == 422
        mock_email.assert_not_called()
        mock_analytics.assert_not_called()
```

```
┌─────────────────────────────────────────────────────────────────┐
│         TestClient AND BACKGROUND TASKS — THE KEY FACT          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IN PRODUCTION:                                                 │
│  1. Response is sent to client (201 Created)                    │
│  2. THEN background tasks run (client is already gone)          │
│                                                                 │
│  IN TestClient:                                                 │
│  1. Background tasks run SYNCHRONOUSLY within client.post()     │
│  2. Response object is returned AFTER tasks have completed      │
│                                                                 │
│  This means: mock_email.assert_called() works immediately       │
│  after client.post(). You don't need asyncio.sleep() or         │
│  any "wait for the task" logic.                                 │
│                                                                 │
│  ⚠️ THIS DIFFERS FROM PRODUCTION BEHAVIOR.                      │
│  In production, tasks run after the response is sent.           │
│  In TestClient, tasks run before you get the Response object.   │
│  Tests are still valid — the function is called either way.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your endpoint POST /users/register creates a user AND          │
│  schedules a background task `send_welcome_email(user_id, email)`│
│  Write a test that: (1) asserts 201 status, (2) verifies the    │
│  email function was called with the correct user_id (from the   │
│  response body) and the email that was sent in the request.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION

from unittest.mock import patch

def test_registration_sends_welcome_email(client: TestClient):
    with patch("app.send_welcome_email") as mock_email:
        response = client.post(
            "/users/register",
            json={"username": "alice", "email": "alice@example.com", "password": "secret"}
        )

        assert response.status_code == 201
        user_id = response.json()["id"]

        mock_email.assert_called_once_with(
            user_id=user_id,
            email="alice@example.com"
        )
```

---

## 5.9 Database Testing Preview (Bridge to Week 6)

**You're currently using an in-memory `TaskStorage` override. In Week 6, you'll swap it for a real PostgreSQL database via SQLAlchemy. The `dependency_overrides` pattern you learned today is EXACTLY what makes this transition work.**

```
┌─────────────────────────────────────────────────────────────────┐
│         THREE DATABASE TEST ISOLATION STRATEGIES                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STRATEGY 1: IN-MEMORY FAKE (current approach — Week 3-4)      │
│  ── Override get_storage() with fresh TaskStorage()             │
│  ── No PostgreSQL involved                                      │
│  ── Fastest, simplest                                           │
│  ── ✅ Correct for now                                          │
│  ── ❌ Doesn't test SQL queries, constraints, or migrations      │
│                                                                 │
│  STRATEGY 2: TRUNCATE TABLES BETWEEN TESTS (don't do this)     │
│  ── Use a real test database                                    │
│  ── DELETE FROM tasks; between tests                            │
│  ── Slow (DDL between every test)                               │
│  ── Dirty state if a test crashes mid-delete                    │
│  ── ❌ Fragile, slow, wrong                                     │
│                                                                 │
│  STRATEGY 3: TRANSACTION ROLLBACK (✅ Week 6 pattern)          │
│  ── Wrap each test in a database transaction                    │
│  ── Roll back EVERYTHING after the test, pass or fail           │
│  ── Database returns to clean state in microseconds             │
│  ── No deletions, no table truncation                           │
│  ── ✅ Fast, clean, correct                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Preview of the Week 6 pattern (do not implement yet — just read this):**

```python
# PREVIEW — Week 6 will implement this in full
# The key insight: dependency_overrides + yield fixture + rollback

from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine

TEST_DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/test_db"
engine = create_async_engine(TEST_DATABASE_URL)


@pytest.fixture
async def db_session():
    """Each test gets a transaction that ROLLS BACK after the test.
    No matter what the test does — inserts, updates, deletes —
    the database is restored to its exact state before the test ran.
    """
    async with AsyncSession(engine) as session:
        await session.begin()          # Open transaction
        yield session                  # Test runs inside this transaction
        await session.rollback()       # Undo EVERYTHING — clean state restored


@pytest.fixture
def client(db_session: AsyncSession) -> TestClient:
    """Inject the rollback session into the app via dependency_overrides.
    The app thinks it's using its normal DB session.
    It's actually using one that will roll back.
    """
    app.dependency_overrides[get_db] = lambda: db_session
    yield TestClient(app)
    app.dependency_overrides.clear()


# Your tests don't change at all:
def test_create_task(client: TestClient):
    response = client.post("/tasks", json={"title": "DB task"})
    assert response.status_code == 201
    # After this test: the DB row is GONE. Rolled back.
    # The next test starts from a clean database.
```

> "Look at how little changes. You still use `dependency_overrides`. You still have a `client` fixture. The only difference is what you inject — a rolling-back session instead of an in-memory class. That's the power of proper dependency injection."

**The connection between now and Week 6:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  TODAY (Weeks 3-4):                  WEEK 6:                    │
│                                                                 │
│  get_storage()                       get_db()                   │
│       │                                   │                     │
│       ▼                                   ▼                     │
│  TaskStorage()  ←── override ───▶  AsyncSession  ←── override  │
│  (in-memory)          │            (real DB, but  │             │
│                        │             rolls back)  │             │
│                        └────────────────────────────            │
│                           same dependency_overrides pattern      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.10 Debugging Failed Tests

**When a test fails, you need a systematic approach. Staring at the failure message is not an approach.**

**Reading a pytest failure:**

```
FAILED tests/test_tasks.py::test_create_task - AssertionError: assert 200 == 201

E   assert 200 == 201
E    +  where 200 = <Response [200 OK]>.status_code
```

```
┌─────────────────────────────────────────────────────────────────┐
│                ANATOMY OF A pytest FAILURE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FAILED tests/test_tasks.py::test_create_task                  │
│    ▲              ▲                   ▲                         │
│    │              └── file             └── function name        │
│    └── FAILED (vs PASSED, ERROR)                                │
│                                                                 │
│  AssertionError: assert 200 == 201                              │
│                         ▲      ▲                                │
│                         │      └── expected (right side)        │
│                         └──────── actual (left side)            │
│                                                                 │
│  E    +  where 200 = <Response [200 OK]>.status_code            │
│                        ▲                                        │
│                        pytest shows WHERE 200 came from         │
│                        (it was response.status_code)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Debugging flags — use these in order of invasiveness:**

```bash
# 1. Run only the failing test (stop noise from others)
pytest tests/test_tasks.py::test_create_task -v

# 2. Show print() output (suppressed by default)
pytest tests/test_tasks.py::test_create_task -v -s

# 3. Stop at the first failure — don't run the rest
pytest tests/test_tasks.py -x

# 4. Compact traceback — less clutter
pytest tests/test_tasks.py --tb=short

# 5. Full traceback with local variable values
pytest tests/test_tasks.py --tb=long

# 6. Drop into the debugger on failure
pytest tests/test_tasks.py --pdb
```

**Adding diagnostic output:**

```python
def test_create_task_returns_201(client: TestClient):
    response = client.post("/tasks", json={"title": "Debug me"})

    # Add these TEMPORARILY when debugging — remove before committing
    print(f"\nStatus:  {response.status_code}")
    print(f"Body:    {response.json()}")
    print(f"Headers: {dict(response.headers)}")

    assert response.status_code == 201
    # Run: pytest test_tasks.py::test_create_task -v -s
    # The -s flag makes print() visible in the terminal
```

**Using breakpoint() for interactive inspection:**

```python
def test_create_task_returns_201(client: TestClient):
    response = client.post("/tasks", json={"title": "Inspect me"})

    breakpoint()   # Execution pauses here — open interactive debugger

    assert response.status_code == 201

# In the pdb prompt:
# (Pdb) response.status_code    → 200  (the actual value)
# (Pdb) response.json()         → {"id": 1, "title": "Inspect me", ...}
# (Pdb) response.headers        → Headers({"content-type": "application/json"})
# (Pdb) c                       → continue (run remaining assertions)
# (Pdb) q                       → quit test run entirely
```

**Common failure patterns and their causes:**

```
┌─────────────────────────────────────────────────────────────────┐
│         COMMON FAILURE PATTERNS → ROOT CAUSES                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  assert 200 == 201                                              │
│  → Endpoint uses default status 200. Check:                     │
│    @app.post("/tasks", status_code=201)                         │
│                                                                 │
│  assert 422 == 201                                              │
│  → Pydantic rejected your request body. Check:                  │
│    print(response.json()) — read the validation errors           │
│    Your json={} payload likely violates a Field constraint       │
│                                                                 │
│  KeyError: 'id'                                                 │
│  → response.json() is an error body, not a task body. Check:    │
│    print(response.json()) BEFORE asserting fields               │
│                                                                 │
│  assert 2 == 1  (in test expecting id==1)                        │
│  → State leaked from a previous test. Fix:                      │
│    Use dependency_overrides with fresh storage per test          │
│                                                                 │
│  fixture 'client' not found                                     │
│  → conftest.py not in the right directory, or                   │
│    fixture scope mismatch. Check file paths.                    │
│                                                                 │
│  RuntimeError: Event loop is closed                             │
│  → asyncio_mode not set in pyproject.toml, or                   │
│    mixing sync and async fixtures incorrectly                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│  ✏️  PRACTICE CHECKPOINT                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A test fails with: assert 422 == 201. Without adding print()   │
│  or a debugger, name THREE things you would check first — and   │
│  for each one, state which tool or command you'd use.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ✅ SOLUTION

# 1. Read the 422 body — add print(response.json()) and run with -s
#    pytest test_tasks.py::my_test -v -s
#    The output shows which field failed validation and why.

# 2. Check your json={} payload against the Pydantic model.
#    Did you send {"title": ""} when min_length=1?
#    Did you omit a required field?

# 3. Check if the field type is wrong.
#    {"priority": "high"} → fails int coercion → 422
#    Run the test with --tb=long to see which line caused the assertion.
#    Use breakpoint() to inspect response.json()["detail"][0]["loc"]
#    to find exactly which field failed.
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                FASTAPI TESTING QUICK REFERENCE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SETUP:                                                         │
│      from fastapi.testclient import TestClient                  │
│      from app import app                                        │
│      client = TestClient(app)               # no lifespan       │
│      with TestClient(app) as client: ...    # with lifespan     │
│                                                                 │
│  MAKE REQUESTS:                                                 │
│      client.get("/path")                                        │
│      client.post("/path", json={...})       # application/json  │
│      client.post("/path", data={...})       # form-urlencoded   │
│      client.post("/path", files={"f": ("name", bytes, "mime")}) │
│      client.post("/path", files={...}, data={...}) # file+form  │
│      client.put("/path", json={...})                            │
│      client.patch("/path", json={...})                          │
│      client.delete("/path")                                     │
│      client.get("/path", params={"key": "val"})                 │
│      client.get("/path", headers={"X-Key": "val"})              │
│      client.get("/path", cookies={"session": "token"})          │
│      client.get("/path", follow_redirects=False) # test 307/308 │
│                                                                 │
│  CHECK RESPONSES:                                               │
│      response.status_code          # 200, 201, 404, 422...      │
│      response.json()               # dict or list               │
│      response.text                 # raw string body            │
│      response.headers              # response headers dict      │
│      response.headers["location"]  # redirect target            │
│      response.cookies              # cookies set by server      │
│      response.cookies["key"]       # specific cookie value      │
│                                                                 │
│  DEPENDENCY OVERRIDES:                                          │
│      app.dependency_overrides[original] = lambda: replacement   │
│      app.dependency_overrides[get_current_user] = lambda: user  │
│      app.dependency_overrides.clear()     # always clean up!    │
│                                                                 │
│  ASYNC TESTING:                                                 │
│      from httpx import AsyncClient, ASGITransport               │
│      transport = ASGITransport(app=app)                         │
│      async with AsyncClient(                                    │
│          transport=transport, base_url="http://test"            │
│      ) as client:                                               │
│          response = await client.get("/path")                   │
│                                                                 │
│  CONCURRENT TESTING:                                            │
│      responses = await asyncio.gather(                          │
│          *[async_client.post("/path", json={...})               │
│            for i in range(10)]                                  │
│      )                                                          │
│                                                                 │
│  MOCKING EXTERNAL HTTP CALLS:                                   │
│      import respx, httpx                                        │
│      @respx.mock                                                │
│      def test_x(client):                                        │
│          respx.get("https://api.example.com/...").mock(         │
│              return_value=httpx.Response(200, json={...})       │
│          )                                                      │
│                                                                 │
│  BACKGROUND TASKS:                                              │
│      from unittest.mock import patch                            │
│      with patch("app.my_task_fn") as mock_task:                 │
│          client.post("/endpoint", json={...})                   │
│          mock_task.assert_called_once_with(arg=value)           │
│                                                                 │
│  COVERAGE:                                                      │
│      pytest --cov=app --cov-report=term-missing                 │
│      pytest --cov=app --cov-report=html                         │
│      pytest --cov=app --cov-fail-under=80                       │
│                                                                 │
│  DEBUGGING:                                                     │
│      pytest test_tasks.py::test_x -v -s    # show print()      │
│      pytest test_tasks.py -x               # stop at first fail │
│      pytest test_tasks.py --pdb            # drop into debugger │
│      breakpoint()   # inside test — interactive pdb             │
│                                                                 │
│  PYTEST-ASYNCIO CONFIG (pyproject.toml):                        │
│      [tool.pytest.ini_options]                                  │
│      asyncio_mode = "auto"     # no @mark.asyncio needed        │
│      # asyncio_mode = "strict" # must mark each async test      │
│                                                                 │
│  COMMON ASSERTIONS:                                             │
│      assert response.status_code == 201                         │
│      assert response.json()["field"] == "value"                 │
│      assert "id" in response.json()                             │
│      assert isinstance(response.json(), list)                   │
│      assert len(response.json()["detail"]) > 0   # 422 errors  │
│      assert "x-request-id" in response.headers                  │
│      assert "refresh_token" in response.cookies                 │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ Only asserting body, not status_code                    │
│      ❌ Not clearing dependency_overrides after test             │
│      ❌ Sharing state between tests (no isolation)              │
│      ❌ Only testing the happy path (skipping 422, 404)         │
│      ❌ Using AsyncClient when TestClient is sufficient          │
│      ❌ Testing implementation details instead of HTTP contract │
│      ❌ Not overriding get_current_user for protected endpoints  │
│      ❌ Making real network calls in tests (use respx)          │
│      ❌ TestClient(app) instead of with TestClient(app)          │
│         when endpoint reads from app.state                      │
│      ❌ Treating 80% coverage as a ceiling instead of a floor   │
│      ❌ Missing asyncio_mode in pyproject.toml (async tests     │
│         silently never run)                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  API TESTING = TESTING THE HTTP CONTRACT                        │
│                                                                 │
│  Your clients don't call Python functions.                      │
│  They send HTTP requests. Test what they experience.            │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                   │
│  │ Request  │ ──▶│ FastAPI  │ ──▶│ Response │                   │
│  │ (JSON,   │    │ Pipeline │    │ (Status, │                   │
│  │  form,   │    │ (FULL)   │    │  JSON,   │                   │
│  │  files,  │    │          │    │  cookies,│                   │
│  │  cookies)│    │          │    │  headers)│                   │
│  └──────────┘    └──────────┘    └──────────┘                   │
│       ▲                               │                         │
│       │          TestClient           │                         │
│       └───────────────────────────────┘                         │
│                                                                 │
│                                                                 │
│  THE VENDING MACHINE ANALOGY — EXTENDED:                        │
│  ├─ TestClient      = testing panel on the back                 │
│  ├─ Request         = inserting coins + pressing a button        │
│  ├─ Response        = what drops out + what the display says     │
│  ├─ dep. overrides  = loading known test products                │
│  ├─ Auth override   = using a master key (bypass coin slot)      │
│  ├─ respx mock      = simulating the supplier without ordering   │
│  ├─ BackgroundTasks = the restocking crew (runs after you leave) │
│  └─ Coverage report = inspection checklist of every button       │
│       (tells you which buttons you forgot to press in testing)   │
│                                                                 │
│                                                                 │
│  SIX NON-NEGOTIABLE RULES:                                     │
│  1. Always assert status codes (not just body)                  │
│  2. Always test rejection (422, 404) — not just success         │
│  3. Always isolate state (dependency_overrides or fixtures)     │
│  4. Test BOTH auth and no-auth paths on protected endpoints     │
│  5. Never call real external APIs — mock them (respx/mock)      │
│  6. 80% coverage is a FLOOR, not a goal                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Connection to Upcoming Lectures

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 4, LECTURE 2 (next):                                      │
│  └─ API Design Principles (versioning, pagination, filtering)   │
│     You'll write tests for these patterns immediately.          │
│     Cursor pagination? Test the cursor. API versioning?         │
│     Test that /v1/tasks and /v2/tasks behave differently.       │
│                                                                 │
│  WEEK 5-6 (Databases):                                          │
│  └─ dependency_overrides becomes CRITICAL.                      │
│     You'll override get_db_session() to provide a test          │
│     database session that ROLLS BACK after each test.           │
│     Same pattern you learned today, higher stakes.              │
│                                                                 │
│  WEEK 8 (External APIs):                                        │
│  └─ Mock external HTTP calls in your tests.                     │
│     Combine dependency_overrides with unittest.mock             │
│     to simulate third-party API responses.                      │
│                                                                 │
│  WEEK 9 (Authentication):                                       │
│  └─ Test protected endpoints. Override get_current_user()       │
│     to simulate authenticated requests without real tokens.     │
│                                                                 │
│  WEEK 12 (WebSockets):                                          │
│  └─ AsyncClient becomes essential.                              │
│     WebSocket testing requires true async — TestClient          │
│     can't simulate bidirectional real-time connections.          │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ Comprehensive test suite: unit + integration, 80%+          │
│     coverage. Everything from today scales up.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```