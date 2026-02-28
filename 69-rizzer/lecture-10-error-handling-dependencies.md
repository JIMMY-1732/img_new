## LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM FIRST, PATTERN SECOND                                  │
│  ─────────────────────────────                                  │
│  Students will HIT a broken API. Ugly 500s, leaked stack        │
│  traces, inconsistent errors. Feel the chaos before the cure.   │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  We use a HOTEL analogy throughout. Error handling is the       │
│  front desk. Dependencies are room services. Logging is the     │
│  hotel logbook.                                                 │
│                                                                 │
│  BUILD INCREMENTALLY                                            │
│  ────────────────────                                           │
│  One broken API. We fix it piece by piece through the lecture.  │
│  Each Part adds a layer. By the end: a clean, observable API.   │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Custom exceptions (W1 L2) → API exception hierarchies          │
│  Context managers (W1 L2) → Yield dependencies                  │
│  Decorators (W1 L2) → @app.exception_handler                    │
│  Type hints (W1 L1) → Typed dependency signatures               │
│  Status codes (W2 L1) → Now we USE them properly                │
│  Pydantic (W2 L3) → Error response models                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                 ERROR HANDLING & DEPENDENCIES                   │
│                     (4-4.5 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (35 min)                                   │
│  ├─ 1.1 The Broken API (Demonstration)                          │
│  ├─ 1.2 What Clients Actually See                               │
│  ├─ 1.3 The Copy-Paste Epidemic                                 │
│  └─ 1.4 The Hotel Analogy                                       │
│                                                                 │
│  PART 2: ERROR HANDLING (55 min)                                │
│  ├─ 2.1 HTTPException — The Structured Error                    │
│  ├─ 2.2 Choosing Status Codes (Connection to Lecture 1)         │
│  ├─ 2.3 Custom Exception Classes (Connection to Week 1 L2)      │
│  ├─ 2.4 Custom Exception Handlers (The Global Safety Net)       │
│  └─ 2.5 Consistent Error Response Schema (Connection to L3)     │
│                                                                 │
│  PART 3: DEPENDENCY INJECTION (90 min)                          │
│  ├─ 3.1 What is Dependency Injection?                           │
│  ├─ 3.2 Depends() — Your First Dependency                       │
│  ├─ 3.3 Dependencies with Return Values                         │
│  ├─ 3.4 Chaining Dependencies                                   │
│  ├─ 3.5 Class-Based Dependencies                                │
│  ├─ 3.6 The Annotated Pattern & Dependency Type Aliases         │
│  ├─ 3.7 Sync vs. Async Dependencies                             │
│  ├─ 3.8 Decorator-Based Dependencies                            │
│  ├─ 3.9 Router and Application-Level Dependencies               │
│  └─ 3.10 Testing with Dependency Overrides                      │
│                                                                 │
│  PART 4: LIFECYCLE & LOGGING (45 min)                           │
│  ├─ 4.1 Yield Dependencies (Connection to Week 1 L2)            │
│  ├─ 4.2 Dependency Scope and Caching                            │
│  ├─ 4.3 Why print() Will Betray You                             │
│  └─ 4.4 structlog Introduction                                  │
│                                                                 │
│  PART 5: COMMON MISTAKES AND BEST PRACTICES (25 min)            │
│  ├─ 5.1 Error Handling Mistakes                                 │
│  ├─ 5.2 Dependency Injection Mistakes                           │
│  └─ 5.3 The Decision Cheatsheet                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Broken API

**Start with a demonstration. Let them hit the wall.**

```python
# broken_api.py — Run this with students watching
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

# Simple in-memory store
tasks_db: dict[int, dict] = {
    1: {"id": 1, "title": "Buy groceries", "completed": False},
    2: {"id": 2, "title": "Write report", "completed": True},
}
next_id: int = 3


class TaskCreate(BaseModel):
    title: str
    completed: bool = False


@app.get("/tasks/{task_id}")
async def get_task(task_id: int):
    return tasks_db[task_id]  # What if task_id doesn't exist?


@app.get("/tasks")
async def list_tasks(skip: int = 0, limit: int = 10):
    if skip < 0:
        return {"error": "skip must be non-negative"}  # Hmmm...
    if limit < 1 or limit > 100:
        return {"error": "limit must be between 1 and 100"}
    all_tasks = list(tasks_db.values())
    return all_tasks[skip : skip + limit]


@app.post("/tasks")
async def create_task(task: TaskCreate):
    global next_id
    # Check for duplicate title
    for existing in tasks_db.values():
        if existing["title"] == task.title:
            return {"error": "Task with this title already exists"}  # Hmmm...
    new_task = {"id": next_id, "title": task.title, "completed": task.completed}
    tasks_db[next_id] = new_task
    next_id += 1
    return new_task


@app.delete("/tasks/{task_id}")
async def delete_task(task_id: int):
    if task_id not in tasks_db:
        return {"error": "Task not found"}  # Hmmm...
    del tasks_db[task_id]
    return {"message": "Deleted"}
```

**Run it. Hit the endpoints.**

```bash
uvicorn broken_api:app --reload
```

---

## 1.2 What Clients Actually See

**Request 1: GET a task that doesn't exist**

```bash
curl http://localhost:8000/tasks/999
```

```json
{
    "detail": "Internal Server Error"
}
```

Status: **500 Internal Server Error**

**Meanwhile, in your terminal:**

```
ERROR:    Exception in ASGI application
Traceback (most recent call last):
  File "broken_api.py", line 18, in get_task
    return tasks_db[task_id]
KeyError: 999
```

> "The client gets a useless `Internal Server Error`. Your terminal leaks the exact line of code that failed. In production, this is a **security risk** and a **debugging nightmare**."

**Request 2: DELETE a task that doesn't exist**

```bash
curl http://localhost:8000/tasks/999
```

```json
{
    "error": "Task not found"
}
```

Status: **200 OK**

**Now ask the class:**

> "Both requests ask for a task that doesn't exist. One returns 500 with a cryptic message. The other returns 200 with an error in the body. You're a frontend developer consuming this API. How do you check if something went wrong?"

Answer: **You can't. There's no consistent contract. Sometimes the status code tells you, sometimes the body does, sometimes neither.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE INCONSISTENCY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GET /tasks/999                                                 │
│  ├─ Status: 500                                                 │
│  └─ Body: {"detail": "Internal Server Error"}                   │
│                                                                 │
│  DELETE /tasks/999                                               │
│  ├─ Status: 200 ← WRONG! This is a success code!               │
│  └─ Body: {"error": "Task not found"}                           │
│                                                                 │
│  POST /tasks (duplicate title)                                  │
│  ├─ Status: 200 ← WRONG AGAIN!                                 │
│  └─ Body: {"error": "Task with this title already exists"}      │
│                                                                 │
│  GET /tasks?skip=-5                                             │
│  ├─ Status: 200 ← STILL WRONG!                                 │
│  └─ Body: {"error": "skip must be non-negative"}                │
│                                                                 │
│  CLIENT CODE:                                                   │
│                                                                 │
│  response = await fetch("/tasks/999")                           │
│  if (response.ok) { ← True for the DELETE (200)!               │
│      // But it's actually an error...                           │
│      // Do I check response.body.error? response.body.detail?   │
│      // Every endpoint returns a different shape!               │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 The Copy-Paste Epidemic

**Look at the code again. Find the duplication.**

```python
# Endpoint 1: list tasks
@app.get("/tasks")
async def list_tasks(skip: int = 0, limit: int = 10):
    if skip < 0:
        return {"error": "skip must be non-negative"}
    if limit < 1 or limit > 100:
        return {"error": "limit must be between 1 and 100"}
    all_tasks = list(tasks_db.values())
    return all_tasks[skip : skip + limit]


# Endpoint 2: list categories (imagine we add this)
@app.get("/categories")
async def list_categories(skip: int = 0, limit: int = 10):
    if skip < 0:
        return {"error": "skip must be non-negative"}  # ← COPY
    if limit < 1 or limit > 100:
        return {"error": "limit must be between 1 and 100"}  # ← PASTE
    all_categories = list(categories_db.values())
    return all_categories[skip : skip + limit]


# Endpoint 3: list users (and another one)
@app.get("/users")
async def list_users(skip: int = 0, limit: int = 10):
    if skip < 0:
        return {"error": "skip must be non-negative"}  # ← COPY
    if limit < 1 or limit > 100:
        return {"error": "limit must be between 1 and 100"}  # ← PASTE
    all_users = list(users_db.values())
    return all_users[skip : skip + limit]
```

**Ask the class:**

> "What happens when your team decides to change the max limit from 100 to 50? How many places do you need to update?"

Answer: **Every. Single. Endpoint.** Miss one? Bug. And it's silent — no error, just wrong behavior.

```
┌─────────────────────────────────────────────────────────────────┐
│                 THREE PROBLEMS TO SOLVE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM 1: UGLY, DANGEROUS ERRORS                              │
│  ├─ Unhandled exceptions → 500 with stack trace                 │
│  ├─ Security risk (leaks internals)                             │
│  └─ Useless for clients                                         │
│                                                                 │
│  PROBLEM 2: INCONSISTENT ERROR FORMAT                           │
│  ├─ Some errors return 200 OK (wrong!)                          │
│  ├─ Some use {"error": ...}, some use {"detail": ...}           │
│  └─ Client can't write reliable error handling code             │
│                                                                 │
│  PROBLEM 3: DUPLICATED LOGIC                                    │
│  ├─ Same validation in every endpoint                           │
│  ├─ One change → update N places                                │
│  └─ Bugs hide in the copies you forgot                          │
│                                                                 │
│  TODAY WE FIX ALL THREE.                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 The Hotel Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE HOTEL ANALOGY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your API is a HOTEL.                                           │
│  Clients are GUESTS.                                            │
│  Endpoints are HOTEL SERVICES (room service, spa, concierge).   │
│                                                                 │
│                                                                 │
│  THE TERRIBLE HOTEL (Our Broken API)                            │
│  ────────────────────────────────────                           │
│                                                                 │
│  Guest: "I'd like room 999."                                    │
│  Staff: "RUNTIME ERROR. KEY ERROR. TRACEBACK LINE 18.           │
│          FILE BROKEN_API.PY. DICT TASKS_DB."                    │
│  Guest: 😱                                                      │
│                                                                 │
│  Guest: "Cancel my reservation for room 999."                   │
│  Staff: "Done!" (smiles, gives thumbs up)                       │
│  Guest: "But I don't HAVE room 999..."                          │
│  Staff: "Oh yeah, we said 'done' but meant 'error'. Sorry."    │
│  Guest: 😤                                                      │
│                                                                 │
│                                                                 │
│  THE PROFESSIONAL HOTEL (What We'll Build)                      │
│  ─────────────────────────────────────────                      │
│                                                                 │
│  Guest: "I'd like room 999."                                    │
│  Front Desk: "I'm sorry, room 999 doesn't exist.               │
│               Here are available rooms: 1, 2, 3."               │
│  Guest: "Ah, I'll take room 2."                                 │
│  Front Desk: ✅ (clear, professional, helpful)                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to what we'll learn:**

```
Hotel                       │  FastAPI
────────────────────────────│────────────────────────────
Front desk complaint        │  HTTPException
  protocol                  │    (structured error responses)
                            │
Complaint manual            │  Custom exception handlers
  (same format every time)  │    (consistent error format)
                            │
Hotel room services         │  Dependencies with Depends()
  (WiFi, towels, minibar    │    (shared resources injected
   ready when you arrive)   │     into every endpoint)
                            │
Room setup / checkout       │  Yield dependencies
  (prepare room before      │    (setup before request,
   guest, clean up after)   │     teardown after)
                            │
Hotel logbook               │  structlog
  (record every guest       │    (structured, searchable
   interaction for review)  │     logging for debugging)
```

---

# PART 2: ERROR HANDLING

## 2.1 HTTPException — The Structured Error

**FastAPI gives you `HTTPException` — the front desk protocol.**

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

tasks_db: dict[int, dict] = {
    1: {"id": 1, "title": "Buy groceries", "completed": False},
    2: {"id": 2, "title": "Write report", "completed": True},
}


@app.get("/tasks/{task_id}")
async def get_task(task_id: int):
    if task_id not in tasks_db:
        raise HTTPException(
            status_code=404,
            detail=f"Task with id {task_id} not found"
        )
    return tasks_db[task_id]
```

**Now hit it:**

```bash
curl http://localhost:8000/tasks/999
```

```json
{
    "detail": "Task with id 999 not found"
}
```

Status: **404 Not Found**

> "The status code is correct (404). The message is clear. The client knows EXACTLY what went wrong. No stack trace, no leaked internals."

**HTTPException is a Python exception — you RAISE it, not RETURN it:**

```
┌─────────────────────────────────────────────────────────────────┐
│               RAISE vs RETURN                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ RETURN an error (what we did before):                       │
│                                                                 │
│     return {"error": "Task not found"}                          │
│     │                                                           │
│     ├─ Status code: 200 OK (WRONG!)                             │
│     ├─ FastAPI thinks this is a SUCCESS response                │
│     └─ Client's response.ok → True (MISLEADING!)               │
│                                                                 │
│                                                                 │
│  ✅ RAISE an exception (the correct way):                       │
│                                                                 │
│     raise HTTPException(status_code=404, detail="Not found")    │
│     │                                                           │
│     ├─ Status code: 404 Not Found (CORRECT!)                   │
│     ├─ FastAPI catches it, formats the response                 │
│     ├─ Code below the raise NEVER executes                     │
│     └─ Client's response.ok → False (ACCURATE!)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What `raise HTTPException` does under the hood:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Your Endpoint            FastAPI              Client           │
│  ─────────────            ───────              ──────           │
│       │                                                         │
│   raise HTTPException ──▶ Catches the         HTTP Response     │
│     (status_code=404,     exception     ──▶   Status: 404      │
│      detail="...")        │                   Body: {"detail":  │
│       │                   Formats it           "..."}           │
│       │                   into HTTP                             │
│       ▼                   response                              │
│   Code stops here.                                              │
│   Nothing below                                                 │
│   this line runs.                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to Week 1, Lecture 2:**

> "Remember `raise` from error handling? Same mechanism. `HTTPException` inherits from Python's `Exception`. When you `raise` it, Python's normal exception propagation kicks in — FastAPI has a handler that catches it and converts it into an HTTP response."

**You can also add custom headers:**

```python
raise HTTPException(
    status_code=429,
    detail="Too many requests",
    headers={"Retry-After": "60"}  # Tell client when to retry
)
```

---

## 2.2 Choosing Status Codes

**Connection to Lecture 1 (HTTP & REST Foundations):**

> "You learned what status codes mean in Lecture 1. Now we use them. This is where knowing the semantics actually matters."

```
┌─────────────────────────────────────────────────────────────────┐
│              STATUS CODE DECISION MAP                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "Is this an error?"                                            │
│       │                                                         │
│      NO ──▶ 2xx SUCCESS                                         │
│       │     ├─ 200: Here's your data                            │
│       │     ├─ 201: Created a new thing                         │
│       │     └─ 204: Done, nothing to show (delete)              │
│       │                                                         │
│      YES                                                        │
│       │                                                         │
│  "Whose fault is it?"                                           │
│       │                                                         │
│   CLIENT'S ──▶ 4xx CLIENT ERROR                                 │
│       │        ├─ 400: Your request doesn't make sense          │
│       │        ├─ 401: Who are you? (not authenticated)         │
│       │        ├─ 403: I know who you are, but NO (forbidden)   │
│       │        ├─ 404: That thing doesn't exist                 │
│       │        ├─ 409: Conflicts with current state (duplicate) │
│       │        └─ 422: I understand your request but the data   │
│       │                 is invalid (Pydantic validation)        │
│       │                                                         │
│   SERVER'S ──▶ 5xx SERVER ERROR                                 │
│                ├─ 500: We messed up (unhandled exception)       │
│                ├─ 502: Upstream service is broken                │
│                └─ 503: We're temporarily overloaded              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In practice for a Task Manager API:**

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()


@app.get("/tasks/{task_id}")
async def get_task(task_id: int):
    if task_id not in tasks_db:
        raise HTTPException(
            status_code=404,                          # Not Found
            detail=f"Task {task_id} not found"
        )
    return tasks_db[task_id]


@app.post("/tasks", status_code=201)                  # Created
async def create_task(task: TaskCreate):
    for existing in tasks_db.values():
        if existing["title"] == task.title:
            raise HTTPException(
                status_code=409,                      # Conflict
                detail=f"Task with title '{task.title}' already exists"
            )
    # ... create task


@app.delete("/tasks/{task_id}", status_code=204)      # No Content
async def delete_task(task_id: int):
    if task_id not in tasks_db:
        raise HTTPException(
            status_code=404,
            detail=f"Task {task_id} not found"
        )
    del tasks_db[task_id]
    # No return needed for 204
```

**The rule:**

> "Status codes are not decoration. They are a CONTRACT with your client. A 404 means 'this doesn't exist'. A 409 means 'this conflicts with something'. The client's `if (response.status === 409)` logic DEPENDS on you choosing correctly."

---

## 2.3 Custom Exception Classes

**Connection to Week 1, Lecture 2:**

> "Remember custom exceptions and exception hierarchies from Week 1? We built `APIError`, `RateLimitError`, `NotFoundError`. Now we do the same thing — but we keep them SEPARATE from HTTP concerns."

**Why not just use HTTPException everywhere?**

```python
# ❌ This works, but has a problem:
async def get_task(task_id: int):
    task = find_task_in_db(task_id)
    if task is None:
        raise HTTPException(status_code=404, detail="Not found")
    return task

# What if find_task_in_db is used by OTHER code, not just endpoints?
# What if a background job calls it? A CLI tool?
# HTTPException is a WEB concept. Your business logic shouldn't
# know about HTTP status codes.
```

```
┌─────────────────────────────────────────────────────────────────┐
│           SEPARATION OF CONCERNS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BUSINESS LOGIC (doesn't know about HTTP)                       │
│  ──────────────                                                 │
│  raise TaskNotFoundError(task_id=99)                            │
│  raise DuplicateTaskError(title="Buy groceries")                │
│                                                                 │
│       │                                                         │
│       │  Exceptions bubble up                                   │
│       ▼                                                         │
│                                                                 │
│  API LAYER (translates to HTTP)                                 │
│  ─────────                                                      │
│  TaskNotFoundError   → 404 Not Found                            │
│  DuplicateTaskError  → 409 Conflict                             │
│  PermissionError     → 403 Forbidden                            │
│  ValueError          → 400 Bad Request                          │
│                                                                 │
│  This separation means your business logic is REUSABLE.         │
│  It doesn't care if it's called from a web API, a CLI,          │
│  or a background job.                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Build the exception hierarchy (Week 1 L2 callback):**

```python
# exceptions.py — Your application's exception hierarchy

class TaskServiceError(Exception):
    """Base exception for task service"""
    pass


class TaskNotFoundError(TaskServiceError):
    """Raised when a task doesn't exist"""
    def __init__(self, task_id: int):
        self.task_id = task_id
        super().__init__(f"Task {task_id} not found")


class DuplicateTaskError(TaskServiceError):
    """Raised when creating a task with a duplicate title"""
    def __init__(self, title: str):
        self.title = title
        super().__init__(f"Task with title '{title}' already exists")


class InvalidTaskStateError(TaskServiceError):
    """Raised when a task state transition is invalid"""
    def __init__(self, task_id: int, current_state: str, requested_state: str):
        self.task_id = task_id
        self.current_state = current_state
        self.requested_state = requested_state
        super().__init__(
            f"Cannot change task {task_id} from '{current_state}' "
            f"to '{requested_state}'"
        )
```

```
┌─────────────────────────────────────────────────────────────────┐
│              EXCEPTION HIERARCHY                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TaskServiceError (base)                                        │
│  ├─ TaskNotFoundError                                           │
│  ├─ DuplicateTaskError                                          │
│  └─ InvalidTaskStateError                                       │
│                                                                 │
│  Same pattern as Week 1 Lecture 2:                              │
│                                                                 │
│  APIError (base)                                                │
│  ├─ RateLimitError                                              │
│  └─ NotFoundError                                               │
│                                                                 │
│  Catch the BASE to handle ALL.                                  │
│  Catch SPECIFIC for precise control.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now use them in your service layer:**

```python
# service.py — Business logic (NO HTTP concepts here!)

def get_task(task_id: int) -> dict:
    """Get a task by ID. Raises TaskNotFoundError if missing."""
    if task_id not in tasks_db:
        raise TaskNotFoundError(task_id)  # Not HTTPException!
    return tasks_db[task_id]


def create_task(title: str, completed: bool = False) -> dict:
    """Create a task. Raises DuplicateTaskError if title exists."""
    for existing in tasks_db.values():
        if existing["title"] == title:
            raise DuplicateTaskError(title)  # Not HTTPException!
    global next_id
    task = {"id": next_id, "title": title, "completed": completed}
    tasks_db[next_id] = task
    next_id += 1
    return task
```

> "Notice: `service.py` doesn't import FastAPI at all. It raises Python exceptions. It's pure business logic. Now we need something to TRANSLATE these into HTTP responses..."

---

## 2.4 Custom Exception Handlers (The Global Safety Net)

**Connection to Week 1, Lecture 2 (Decorators):**

> "Remember decorators from Week 1? `@app.exception_handler` is a decorator that registers a function to handle a specific exception type globally. Every time that exception is raised anywhere in your app, FastAPI routes it to your handler."

```python
# main.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from exceptions import TaskNotFoundError, DuplicateTaskError, TaskServiceError

app = FastAPI()


@app.exception_handler(TaskNotFoundError)
async def task_not_found_handler(request: Request, exc: TaskNotFoundError):
    return JSONResponse(
        status_code=404,
        content={
            "error": "not_found",
            "detail": str(exc),
            "task_id": exc.task_id,
        }
    )


@app.exception_handler(DuplicateTaskError)
async def duplicate_task_handler(request: Request, exc: DuplicateTaskError):
    return JSONResponse(
        status_code=409,
        content={
            "error": "conflict",
            "detail": str(exc),
            "title": exc.title,
        }
    )


# Catch-all for any TaskServiceError we didn't handle specifically
@app.exception_handler(TaskServiceError)
async def task_service_error_handler(request: Request, exc: TaskServiceError):
    return JSONResponse(
        status_code=400,
        content={
            "error": "bad_request",
            "detail": str(exc),
        }
    )
```

**Now your endpoints are clean:**

```python
from service import get_task, create_task

@app.get("/tasks/{task_id}")
async def read_task(task_id: int):
    return get_task(task_id)  # If not found, TaskNotFoundError → 404
    # No try/except needed here!
    # The exception handler catches it globally.


@app.post("/tasks", status_code=201)
async def make_task(task: TaskCreate):
    return create_task(task.title, task.completed)  # Duplicate → 409
    # Clean. Readable. No error handling clutter.
```

**How the flow works:**

```
┌─────────────────────────────────────────────────────────────────┐
│              EXCEPTION HANDLER FLOW                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client Request                                                 │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────┐     ┌──────────────┐     ┌───────────────────┐   │
│  │ Endpoint │ ──▶ │ Service      │ ──▶ │ raise             │   │
│  │ read_task│     │ get_task()   │     │ TaskNotFoundError │   │
│  └──────────┘     └──────────────┘     └─────────┬─────────┘   │
│                                                   │             │
│                                        Exception bubbles up     │
│                                                   │             │
│                                                   ▼             │
│                                        ┌───────────────────┐   │
│                                        │ FastAPI checks:   │   │
│                                        │ "Do I have a      │   │
│                                        │  handler for      │   │
│                                        │  TaskNotFound     │   │
│                                        │  Error?"          │   │
│                                        └─────────┬─────────┘   │
│                                                   │             │
│                                                  YES            │
│                                                   │             │
│                                                   ▼             │
│                                        ┌───────────────────┐   │
│                                        │ task_not_found_   │   │
│                                        │ handler()         │   │
│                                        │ → JSONResponse    │   │
│                                        │   status=404      │   │
│                                        └─────────┬─────────┘   │
│                                                   │             │
│                                                   ▼             │
│                                        Client gets clean 404    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Back to the hotel analogy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  EXCEPTION HANDLER = FRONT DESK COMPLAINT PROTOCOL              │
│                                                                 │
│  Without handler:                                               │
│  Kitchen catches fire → Waiter screams raw error at guest       │
│                                                                 │
│  With handler:                                                  │
│  Kitchen catches fire → Front desk intercepts →                 │
│  "We apologize, your meal will be slightly delayed.             │
│   May we offer you a complimentary drink?"                      │
│                                                                 │
│  The GUEST (client) never sees the chaos.                       │
│  The LOGBOOK (logs) records everything for management.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.5 Consistent Error Response Schema

**Connection to Lecture 3 (Pydantic Deep Dive):**

> "Remember Pydantic models from last lecture? We used them for request and response bodies. Now we create a model for ERROR responses too. This guarantees every error has the same shape."

```python
# schemas.py
from pydantic import BaseModel
from typing import Any


class ErrorResponse(BaseModel):
    """Standard error response — every error returns this shape."""
    error: str           # Machine-readable error code: "not_found"
    detail: str          # Human-readable message: "Task 99 not found"
    field: str | None = None  # Which field caused it (for validation)

    model_config = {"json_schema_extra": {
        "examples": [
            {
                "error": "not_found",
                "detail": "Task 99 not found",
                "field": None,
            }
        ]
    }}
```

**Now update your exception handlers to always return this shape:**

```python
from schemas import ErrorResponse


@app.exception_handler(TaskNotFoundError)
async def task_not_found_handler(request: Request, exc: TaskNotFoundError):
    return JSONResponse(
        status_code=404,
        content=ErrorResponse(
            error="not_found",
            detail=str(exc),
        ).model_dump()
    )


@app.exception_handler(DuplicateTaskError)
async def duplicate_task_handler(request: Request, exc: DuplicateTaskError):
    return JSONResponse(
        status_code=409,
        content=ErrorResponse(
            error="conflict",
            detail=str(exc),
            field="title",
        ).model_dump()
    )
```

**The client contract is now CONSISTENT:**

```
┌─────────────────────────────────────────────────────────────────┐
│              BEFORE vs AFTER                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE (every endpoint returns different error shapes):        │
│                                                                 │
│  GET /tasks/999   → {"detail": "Internal Server Error"}         │
│  DELETE /tasks/999 → {"error": "Task not found"}                │
│  POST /tasks      → {"error": "Task with this title..."}       │
│  POST /tasks      → {"detail": [{"loc": [...], "msg":...}]}    │
│                                                                 │
│  🤯 4 endpoints, 4 different shapes                              │
│                                                                 │
│                                                                 │
│  AFTER (every error returns the same shape):                    │
│                                                                 │
│  GET /tasks/999    → {"error":"not_found",                      │
│                       "detail":"Task 99 not found",             │
│                       "field": null}                            │
│                                                                 │
│  POST /tasks       → {"error":"conflict",                       │
│                       "detail":"Task with title '...' exists",  │
│                       "field":"title"}                          │
│                                                                 │
│  ✅ Every error: same shape. Client parses once.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**You can also override FastAPI's built-in validation error handler:**

```python
from fastapi.exceptions import RequestValidationError


@app.exception_handler(RequestValidationError)
async def validation_error_handler(
    request: Request, exc: RequestValidationError
):
    """Override Pydantic validation errors to match our error format."""
    first_error = exc.errors()[0]
    field_path = " → ".join(str(loc) for loc in first_error["loc"])

    return JSONResponse(
        status_code=422,
        content=ErrorResponse(
            error="validation_error",
            detail=first_error["msg"],
            field=field_path,
        ).model_dump()
    )
```

**Before (FastAPI default validation error):**

```json
{
    "detail": [
        {
            "type": "string_type",
            "loc": ["body", "title"],
            "msg": "Input should be a valid string",
            "input": 12345
        }
    ]
}
```

**After (our consistent format):**

```json
{
    "error": "validation_error",
    "detail": "Input should be a valid string",
    "field": "body → title"
}
```

> "Now a frontend developer can write ONE error handling function for ALL your endpoints. They always check `response.error` and `response.detail`. Predictable. Professional."

---

# PART 3: DEPENDENCY INJECTION

## 3.1 What is Dependency Injection?

**The hotel room analogy:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DEPENDENCY INJECTION                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT DEPENDENCY INJECTION:                                  │
│  ─────────────────────────────                                  │
│                                                                 │
│  You check into a hotel room.                                   │
│  It's EMPTY.                                                    │
│  You need towels? Go find the supply closet yourself.           │
│  You need WiFi? Go wire the router yourself.                    │
│  You need soap? Go to the store yourself.                       │
│                                                                 │
│  Every guest does the same work. Every guest might do it wrong. │
│                                                                 │
│                                                                 │
│  WITH DEPENDENCY INJECTION:                                     │
│  ──────────────────────────                                     │
│                                                                 │
│  You check into a hotel room.                                   │
│  It's READY.                                                    │
│  Towels: already there. WiFi: already connected.                │
│  Soap: already stocked. You just USE them.                      │
│                                                                 │
│  The HOTEL (framework) prepared everything.                     │
│  You DECLARED what you need. The hotel PROVIDED it.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In code terms:**

```python
# WITHOUT dependency injection:
# Every endpoint fetches its own resources

@app.get("/tasks")
async def list_tasks(skip: int = 0, limit: int = 10):
    # Validate pagination HERE
    if skip < 0:
        raise HTTPException(status_code=400, detail="skip must be >= 0")
    if limit < 1 or limit > 100:
        raise HTTPException(status_code=400, detail="limit must be 1-100")
    # Use it
    return get_all_tasks()[skip : skip + limit]


@app.get("/categories")
async def list_categories(skip: int = 0, limit: int = 10):
    # Validate pagination AGAIN (copy-paste)
    if skip < 0:
        raise HTTPException(status_code=400, detail="skip must be >= 0")
    if limit < 1 or limit > 100:
        raise HTTPException(status_code=400, detail="limit must be 1-100")
    # Use it
    return get_all_categories()[skip : skip + limit]


# WITH dependency injection:
# "I NEED pagination. Framework, give it to me, already validated."

@app.get("/tasks")
async def list_tasks(pagination = Depends(get_pagination)):
    return get_all_tasks()[pagination.skip : pagination.skip + pagination.limit]


@app.get("/categories")
async def list_categories(pagination = Depends(get_pagination)):
    return get_all_categories()[pagination.skip : pagination.skip + pagination.limit]
```

**The core idea:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  DEPENDENCY INJECTION IN ONE SENTENCE:                          │
│                                                                 │
│  Your endpoint DECLARES what it needs.                          │
│  The framework PROVIDES it.                                     │
│                                                                 │
│  Endpoint says: "I need pagination parameters."                 │
│  FastAPI says:  "Here you go. Already validated."               │
│                                                                 │
│  Endpoint says: "I need the current user."                      │
│  FastAPI says:  "Here's the user. Already authenticated."       │
│                                                                 │
│  Endpoint says: "I need a database session."                    │
│  FastAPI says:  "Here's a session. I'll clean it up after."     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Depends() — Your First Dependency
**Let's fix the copy-paste problem from Part 1.**

```python
from typing import Annotated
from fastapi import FastAPI, Depends, HTTPException, Query

app = FastAPI()


# STEP 1: Define the dependency — it's just a function!
async def get_pagination(
    skip: int = Query(default=0, ge=0, description="Items to skip"),
    limit: int = Query(default=10, ge=1, le=100, description="Max items"),
) -> dict[str, int]:
    """Common pagination parameters. Used by any list endpoint."""
    return {"skip": skip, "limit": limit}
```

**There are two styles for declaring dependencies. You will see BOTH in the wild.**

```python
# LEGACY STYLE (FastAPI < 0.95, still valid, common in older codebases):
@app.get("/tasks")
async def list_tasks_legacy(pagination: dict[str, int] = Depends(get_pagination)):
    return list(tasks_db.values())[pagination["skip"]:pagination["skip"] + pagination["limit"]]


# MODERN STYLE (FastAPI 0.95+, recommended):
@app.get("/tasks")
async def list_tasks(pagination: Annotated[dict[str, int], Depends(get_pagination)]):
    skip = pagination["skip"]
    limit = pagination["limit"]
    all_tasks = list(tasks_db.values())
    return all_tasks[skip : skip + limit]


@app.get("/categories")
async def list_categories(pagination: Annotated[dict[str, int], Depends(get_pagination)]):
    skip = pagination["skip"]
    limit = pagination["limit"]
    all_categories = list(categories_db.values())
    return all_categories[skip : skip + limit]
```

**Why does the modern style exist? The legacy style has a subtle lie in it:**

```
┌─────────────────────────────────────────────────────────────────┐
│             ANNOTATED SYNTAX BREAKDOWN                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LEGACY STYLE — misleading:                                     │
│                                                                 │
│  pagination: dict[str, int] = Depends(get_pagination)           │
│              ─────────────── ─────────────────────────          │
│              Type hint        Looks like a default value.       │
│                               But it's NOT. Depends() is        │
│                               not a real value — it is          │
│                               metadata for FastAPI.             │
│                               Type checkers get confused.        │
│                                                                 │
│                                                                 │
│  MODERN STYLE — honest:                                         │
│                                                                 │
│  pagination: Annotated[dict[str, int], Depends(get_pagination)] │
│              ─────────┬──────────────  ──────────┬────────────  │
│                       │                          │              │
│              The ACTUAL TYPE:           METADATA about          │
│              dict[str, int].            how FastAPI should       │
│              Type checkers              PROVIDE this type.      │
│              see this correctly.        Not a default value.     │
│                                                                 │
│  Annotated says: "This is a dict[str, int], AND it should       │
│  be produced by calling Depends(get_pagination) at runtime."    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What `Depends()` does under the hood — same as before:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 HOW Depends() WORKS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Client sends:   GET /tasks?skip=10&limit=20                 │
│                                                                 │
│  2. FastAPI sees:   Annotated[..., Depends(get_pagination)]     │
│                     "This endpoint needs pagination"            │
│                                                                 │
│  3. FastAPI calls:  get_pagination(skip=10, limit=20)           │
│                     Extracts skip and limit from query params   │
│                     Validates with Query() constraints          │
│                                                                 │
│  4. FastAPI injects: pagination = {"skip": 10, "limit": 20}    │
│                      Passes the RETURN VALUE to your endpoint   │
│                                                                 │
│  5. Your endpoint:  Uses pagination directly                    │
│                     No validation code needed!                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Depends() is not calling the function. It's TELLING FastAPI: "call this function for me and give me the result."**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Annotated[..., Depends(get_pagination)]                        │
│  │                                                              │
│  ├─ Does NOT call get_pagination immediately                    │
│  ├─ TELLS FastAPI: "Before running this endpoint,               │
│  │   call get_pagination with the right parameters"             │
│  └─ The return value becomes the parameter value                │
│                                                                 │
│  Think of it as room service:                                   │
│  Depends(get_pagination) = "Send up the pagination tray"        │
│  You don't go to the kitchen. The hotel brings it to you.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Bonus: Depends auto-generates OpenAPI docs!**

> "Check Swagger UI at /docs. The `skip` and `limit` query parameters appear automatically in every endpoint that uses `Depends(get_pagination)`. FastAPI inspects the dependency's signature and merges it into the endpoint's documentation."

```
┌─────────────────────────────────────────────────────────────────┐
│                 PRACTICE CHECKPOINT 3.2                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The endpoint below uses the legacy style.                      │
│  Rewrite BOTH parameters using the Annotated pattern.           │
│                                                                 │
│  async def read_task(                                           │
│      task_id: int,                                              │
│      log = Depends(get_request_logger),                         │
│      version: str = Depends(get_api_version),                   │
│  ):                                                             │
│      ...                                                        │
│                                                                 │
│  Also: what import is now required at the top of the file?      │
│                                                                 │
│  ─────────────────────────────────── SOLUTION ─────────────────│
│                                                                 │
│  from typing import Annotated                                   │
│                                                                 │
│  async def read_task(                                           │
│      task_id: int,                                              │
│      log: Annotated[BoundLogger, Depends(get_request_logger)],  │
│      version: Annotated[str, Depends(get_api_version)],         │
│  ):                                                             │
│      ...                                                        │
│                                                                 │
│  Notice: task_id: int has no Annotated wrapper because it is    │
│  a plain path parameter, not a dependency injection.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```


---

## 3.3 Dependencies with Return Values

**Using dataclasses for cleaner return types (Week 1 L2 callback):**

```python
from typing import Annotated
from dataclasses import dataclass
from fastapi import Depends, Query


@dataclass
class PaginationParams:
    skip: int
    limit: int


async def get_pagination(
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=10, ge=1, le=100),
) -> PaginationParams:
    return PaginationParams(skip=skip, limit=limit)


# -- Define a type alias ONCE --
PaginationDep = Annotated[PaginationParams, Depends(get_pagination)]


# -- Use it everywhere --
@app.get("/tasks")
async def list_tasks(pagination: PaginationDep):
    all_tasks = list(tasks_db.values())
    return all_tasks[pagination.skip : pagination.skip + pagination.limit]


@app.get("/categories")
async def list_categories(pagination: PaginationDep):
    all_categories = list(categories_db.values())
    return all_categories[pagination.skip : pagination.skip + pagination.limit]
```

**Type hints make this self-documenting (Week 1 L1 callback):**

```python
# Your IDE knows pagination.skip is int
# mypy verifies it at type-check time
# No guessing, no KeyError from dict access
```

**The type alias is the real payoff of the Annotated pattern:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 TYPE ALIAS ADVANTAGE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT TYPE ALIAS:                                            │
│                                                                 │
│  async def list_tasks(                                          │
│      pagination: Annotated[PaginationParams,                    │
│                             Depends(get_pagination)]            │
│  ):                                                             │
│                                                                 │
│  async def list_categories(                                     │
│      pagination: Annotated[PaginationParams,                    │
│                             Depends(get_pagination)]  ← REPEAT  │
│  ):                                                             │
│                                                                 │
│                                                                 │
│  WITH TYPE ALIAS:                                               │
│                                                                 │
│  PaginationDep = Annotated[PaginationParams, Depends(get_pagination)]  │
│  (defined ONCE — in a shared dependencies.py module)            │
│                                                                 │
│  async def list_tasks(pagination: PaginationDep):               │
│  async def list_categories(pagination: PaginationDep):          │
│                                                                 │
│  If you need to swap get_pagination for get_pagination_v2:      │
│  update ONE line. Every endpoint automatically uses the new     │
│  dependency. Zero drift possible.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Dependencies can do more than parse query params:**

```python
from fastapi import Header

# Dependency that reads from a header
async def get_api_version(
    x_api_version: str = Header(default="v1")
) -> str:
    if x_api_version not in ("v1", "v2"):
        raise HTTPException(
            status_code=400,
            detail=f"Unsupported API version: {x_api_version}"
        )
    return x_api_version


# Type alias for this one too
ApiVersionDep = Annotated[str, Depends(get_api_version)]


@app.get("/tasks")
async def list_tasks(
    pagination: PaginationDep,
    api_version: ApiVersionDep,
):
    # Both dependencies resolved before this code runs
    if api_version == "v2":
        return {"version": "v2", "tasks": list(tasks_db.values())}
    return list(tasks_db.values())
```

**An endpoint can have MULTIPLE dependencies — the hotel room comes with towels AND WiFi AND minibar:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  @app.get("/tasks")                                             │
│  async def list_tasks(                                          │
│      pagination: PaginationDep,   ← Towels                      │
│      api_version: ApiVersionDep,  ← WiFi                        │
│      sort_by: SortDep,            ← Minibar                     │
│  ):                                                             │
│      # All three are ready before your code runs                │
│      ...                                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 PRACTICE CHECKPOINT 3.3                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You have this dependency:                                      │
│                                                                 │
│  async def get_request_logger() -> BoundLogger:                 │
│      return logger.bind(request_id=str(uuid.uuid4())[:8])       │
│                                                                 │
│  1. Define a type alias called LogDep for it.                   │
│  2. Write an endpoint that uses BOTH PaginationDep and LogDep.  │
│  3. If your team later needs to change get_request_logger to    │
│     add tracing context, how many endpoint signatures change?   │
│                                                                 │
│  ─────────────────────────────────── SOLUTION ─────────────────│
│                                                                 │
│  LogDep = Annotated[BoundLogger, Depends(get_request_logger)]   │
│                                                                 │
│  @app.get("/tasks")                                             │
│  async def list_tasks(                                          │
│      pagination: PaginationDep,                                 │
│      log: LogDep,                                               │
│  ):                                                             │
│      log.info("listing_tasks", skip=pagination.skip)            │
│      ...                                                        │
│                                                                 │
│  Zero endpoint signatures change. Only the LogDep alias         │
│  definition needs updating. That is the entire point.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

## 3.4 Chaining Dependencies

**Dependencies can depend on OTHER dependencies.**

```
┌─────────────────────────────────────────────────────────────────┐
│              DEPENDENCY CHAIN                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Think of hotel services that depend on other services:         │
│                                                                 │
│  Room Service depends on:                                       │
│  ├─ Kitchen (prepares food)                                     │
│  │   └─ Kitchen depends on Inventory (checks stock)             │
│  └─ Guest Verification (confirms room number)                   │
│                                                                 │
│  Guest says: "Send room service."                               │
│  Hotel resolves the ENTIRE CHAIN automatically.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In code — a preview of what you'll build in Week 5 (Authentication):**

```python
from fastapi import Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer

# Level 1: Extract the token from the request
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


# Level 2: Decode the token → get user data
async def get_current_user(token: str = Depends(oauth2_scheme)) -> dict:
    """Depends on oauth2_scheme to get the token first."""
    user = decode_token(token)  # Imagine this decodes a JWT
    if user is None:
        raise HTTPException(status_code=401, detail="Invalid token")
    return user


# Level 3: Check if user is admin
async def get_admin_user(user: dict = Depends(get_current_user)) -> dict:
    """Depends on get_current_user — which depends on oauth2_scheme."""
    if user.get("role") != "admin":
        raise HTTPException(status_code=403, detail="Admin access required")
    return user


# Your endpoint — just declares the FINAL dependency it needs:
@app.delete("/tasks/{task_id}")
async def delete_task(task_id: int, admin: dict = Depends(get_admin_user)):
    # By the time we get here:
    # 1. Token was extracted from Authorization header ✅
    # 2. Token was decoded and validated ✅
    # 3. User's admin role was verified ✅
    # ALL automatically. Our endpoint just deletes the task.
    del tasks_db[task_id]
```

**Visualize the chain:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DEPENDENCY RESOLUTION ORDER                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client sends: DELETE /tasks/5                                  │
│                Authorization: Bearer eyJhb...                   │
│                                                                 │
│  FastAPI resolves dependencies BOTTOM-UP:                       │
│                                                                 │
│  ┌─────────────────┐                                            │
│  │ oauth2_scheme   │ ← Step 1: Extract token from header       │
│  │ Returns: token  │                                            │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │ get_current_user│ ← Step 2: Decode token, get user           │
│  │ Returns: user   │                                            │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │ get_admin_user  │ ← Step 3: Check admin role                 │
│  │ Returns: admin  │                                            │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │ delete_task()   │ ← Step 4: NOW your endpoint runs           │
│  │ Uses: admin     │                                            │
│  └─────────────────┘                                            │
│                                                                 │
│  If ANY step fails (bad token, not admin), the chain STOPS.     │
│  Your endpoint code NEVER runs. Clean. Safe.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The power of chaining:**

> "Your `delete_task` endpoint is 2 lines of code. But behind the scenes, authentication, authorization, and token validation all happen automatically. You DECLARED `Depends(get_admin_user)`, and FastAPI resolved the entire chain. This is dependency injection at work."

---

## 3.5 Class-Based Dependencies

**For dependencies with multiple parameters, classes are cleaner:**

```python
from fastapi import Query


class PaginationParams:
    """Class-based dependency for pagination."""
    def __init__(
        self,
        skip: int = Query(default=0, ge=0, description="Items to skip"),
        limit: int = Query(default=10, ge=1, le=100, description="Max items"),
    ):
        self.skip = skip
        self.limit = limit


# Notice: Depends() with no argument!
# FastAPI calls the CLASS CONSTRUCTOR as the dependency.
@app.get("/tasks")
async def list_tasks(pagination: PaginationParams = Depends()):
    all_tasks = list(tasks_db.values())
    return all_tasks[pagination.skip : pagination.skip + pagination.limit]
```

**How `Depends()` with no argument works:**

```
┌─────────────────────────────────────────────────────────────────┐
│              FUNCTION vs CLASS DEPENDENCY                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FUNCTION DEPENDENCY:                                           │
│  ────────────────────                                           │
│                                                                 │
│  async def get_pagination(...) -> PaginationParams:             │
│      return PaginationParams(skip=skip, limit=limit)            │
│                                                                 │
│  pagination: PaginationParams = Depends(get_pagination)         │
│                                 ▲                               │
│                                 └── Pass the function           │
│                                                                 │
│                                                                 │
│  CLASS DEPENDENCY:                                              │
│  ─────────────────                                              │
│                                                                 │
│  class PaginationParams:                                        │
│      def __init__(self, skip: int = ..., limit: int = ...):     │
│          self.skip = skip                                       │
│          self.limit = limit                                     │
│                                                                 │
│  pagination: PaginationParams = Depends()                       │
│                                 ▲                               │
│                                 └── No argument! FastAPI looks  │
│                                     at the TYPE HINT and calls  │
│                                     PaginationParams.__init__() │
│                                                                 │
│  Both do the same thing. Class version is more compact          │
│  when the dependency IS the return type.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A more complex class dependency — filtering:**

```python
from typing import Optional
from fastapi import Query


class TaskFilters:
    """Class-based dependency for task filtering."""
    def __init__(
        self,
        completed: Optional[bool] = Query(default=None, description="Filter by status"),
        search: Optional[str] = Query(default=None, min_length=1, max_length=100),
        sort_by: str = Query(default="id", pattern="^(id|title|completed)$"),
    ):
        self.completed = completed
        self.search = search
        self.sort_by = sort_by


@app.get("/tasks")
async def list_tasks(
    pagination: PaginationParams = Depends(),
    filters: TaskFilters = Depends(),
):
    """Two class dependencies — pagination AND filtering."""
    tasks = list(tasks_db.values())

    # Apply filters
    if filters.completed is not None:
        tasks = [t for t in tasks if t["completed"] == filters.completed]
    if filters.search:
        tasks = [t for t in tasks if filters.search.lower() in t["title"].lower()]

    # Apply pagination
    return tasks[pagination.skip : pagination.skip + pagination.limit]
```

**The endpoint is clean.** All parameter extraction, validation, and parsing happens in the dependency classes. The endpoint focuses on **business logic**.
---
## 3.6 The Annotated Pattern & Dependency Type Aliases

**This section consolidates the Annotated pattern into a practical project structure.**

In a real FastAPI project, dependency type aliases live in a dedicated module, not scattered across router files. Here is the recommended structure:

```
app/
├── main.py
├── routers/
│   ├── tasks.py
│   └── categories.py
└── dependencies.py     ← All shared dependency functions AND aliases live here
```

```python
# dependencies.py — the single source of truth for dependency aliases

from typing import Annotated
from dataclasses import dataclass
from fastapi import Depends, Query, Header, HTTPException
import structlog


# --- Pagination ---

@dataclass
class PaginationParams:
    skip: int
    limit: int


async def get_pagination(
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=10, ge=1, le=100),
) -> PaginationParams:
    return PaginationParams(skip=skip, limit=limit)


PaginationDep = Annotated[PaginationParams, Depends(get_pagination)]


# --- Logging ---

logger = structlog.get_logger()


async def get_request_logger() -> structlog.stdlib.BoundLogger:
    import uuid
    return logger.bind(request_id=str(uuid.uuid4())[:8])


LogDep = Annotated[structlog.stdlib.BoundLogger, Depends(get_request_logger)]


# --- API Version ---

async def get_api_version(
    x_api_version: str = Header(default="v1")
) -> str:
    if x_api_version not in ("v1", "v2"):
        raise HTTPException(status_code=400, detail=f"Unsupported version: {x_api_version}")
    return x_api_version


ApiVersionDep = Annotated[str, Depends(get_api_version)]
```

**Now your router files become extremely clean:**

```python
# routers/tasks.py
from fastapi import APIRouter
from dependencies import PaginationDep, LogDep, ApiVersionDep

router = APIRouter(prefix="/tasks", tags=["tasks"])


@router.get("")
async def list_tasks(
    pagination: PaginationDep,
    log: LogDep,
    api_version: ApiVersionDep,
):
    log.info("listing_tasks", skip=pagination.skip, limit=pagination.limit)
    all_tasks = list(tasks_db.values())
    return all_tasks[pagination.skip : pagination.skip + pagination.limit]
```

**The naming convention:**

```
┌─────────────────────────────────────────────────────────────────┐
│             TYPE ALIAS NAMING CONVENTIONS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pattern: <WhatItProvides>Dep                                   │
│                                                                 │
│  PaginationDep  = Annotated[PaginationParams, Depends(...)]     │
│  LogDep         = Annotated[BoundLogger,      Depends(...)]     │
│  ApiVersionDep  = Annotated[str,              Depends(...)]     │
│  CurrentUserDep = Annotated[User,             Depends(...)]     │
│                                         (you'll build this      │
│                                          in Week 5)             │
│                                                                 │
│  The Dep suffix signals to any reader: "this parameter's        │
│  value is injected by FastAPI, not passed by the caller."       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**One more benefit: Annotated aliases work identically with class-based dependencies (from 3.5):**

```python
# Class-based dependency → Depends() with no argument still works inside Annotated
class TaskFilters:
    def __init__(
        self,
        completed: bool | None = Query(default=None),
        search: str | None = Query(default=None, max_length=100),
    ):
        self.completed = completed
        self.search = search


TaskFiltersDep = Annotated[TaskFilters, Depends()]
#                                       ──────────
#                                       No argument: FastAPI uses
#                                       TaskFilters.__init__ as the callable


@router.get("")
async def list_tasks(
    pagination: PaginationDep,
    filters: TaskFiltersDep,  # Clean. No Depends() visible at all.
):
    tasks = list(tasks_db.values())
    if filters.completed is not None:
        tasks = [t for t in tasks if t["completed"] == filters.completed]
    return tasks[pagination.skip : pagination.skip + pagination.limit]
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 PRACTICE CHECKPOINT 3.6                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your team decides to add a request_id field to every log       │
│  entry AND stamp it into every response header.                 │
│  The new logic lives in get_request_logger().                   │
│                                                                 │
│  You have 47 endpoints that use LogDep.                         │
│                                                                 │
│  How many files do you need to change?                          │
│  Which file? Which line?                                        │
│                                                                 │
│  ─────────────────────────────────── SOLUTION ─────────────────│
│                                                                 │
│  One file: dependencies.py                                      │
│  One change: the body of get_request_logger()                   │
│                                                                 │
│  The 47 endpoints using LogDep are untouched.                   │
│  LogDep still resolves to BoundLogger — the type doesn't        │
│  change, only the internal logic of how it's produced.          │
│  This is the single-responsibility principle applied to         │
│  dependency management.                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.7 Sync vs. Async Dependencies

**All dependency examples so far have used `async def`. But there's an important distinction.**

**Connection to Week 1, Lecture 3 (Async Fundamentals):**

> "Remember blocking vs. non-blocking from Week 1? That same distinction applies to dependencies. If your dependency calls a synchronous blocking library — a legacy DB driver, a file system operation, a CPU-heavy task — and you declare it as `async def`, you block the entire event loop."

```python
import time

# ❌ WRONG: async def with a BLOCKING call inside
# This freezes the event loop while time.sleep() runs.
# While this dependency sleeps for 1 second, ZERO other
# requests can be processed.
async def get_config_wrong() -> dict:
    time.sleep(1)            # BLOCKING — freezes the event loop!
    return {"max_tasks": 100}


# ✅ CORRECT: def (not async def) for BLOCKING operations
# FastAPI runs this in a THREADPOOL automatically.
# The event loop stays free to handle other requests
# while this runs in a background thread.
def get_config_correct() -> dict:
    time.sleep(1)            # Blocking, but safely in a threadpool
    return {"max_tasks": 100}


# ✅ ALSO CORRECT: async def for NON-BLOCKING operations
# Uses await — gives up control back to the event loop.
async def get_config_async() -> dict:
    await asyncio.sleep(1)   # Non-blocking — event loop handles others
    return {"max_tasks": 100}
```

```
┌─────────────────────────────────────────────────────────────────┐
│             def vs async def IN DEPENDENCIES                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  def dependency:                                                │
│  ─────────────                                                  │
│  ├─ FastAPI detects it is a regular function                    │
│  ├─ Runs it in a THREADPOOL (separate thread)                   │
│  ├─ Event loop is NOT blocked                                   │
│  └─ Use when: calling sync libs (requests, psycopg2,            │
│     boto3, file I/O, CPU-heavy work)                            │
│                                                                 │
│                                                                 │
│  async def dependency:                                          │
│  ──────────────────────                                         │
│  ├─ FastAPI awaits it directly in the event loop                │
│  ├─ MUST use await for any I/O (asyncpg, httpx, aiofiles)       │
│  ├─ If it calls blocking code without await → event loop FREEZES│
│  └─ Use when: calling async libs (asyncpg, httpx.AsyncClient,   │
│     redis.asyncio, aiofiles)                                    │
│                                                                 │
│                                                                 │
│  THE RULE:                                                      │
│  If it uses await → async def.                                  │
│  If it calls blocking I/O → def (NOT async def).               │
│  If it's pure computation → either works, prefer def.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A practical scenario — integrating a legacy sync library:**

```python
# Suppose you're integrating a sync-only SDK (no async version exists)

import some_legacy_sdk  # Has no async support


# ❌ WRONG: Wrapping sync SDK in async def
async def get_legacy_data(resource_id: int) -> dict:
    # This blocks. The event loop can't serve other requests
    # until some_legacy_sdk.fetch() completes.
    return some_legacy_sdk.fetch(resource_id)


# ✅ CORRECT: Regular def
def get_legacy_data(resource_id: int) -> dict:
    # FastAPI runs this in a threadpool.
    # Other requests are processed normally while this thread works.
    return some_legacy_sdk.fetch(resource_id)


LegacyDataDep = Annotated[dict, Depends(get_legacy_data)]


@app.get("/items/{resource_id}")
async def get_item(data: LegacyDataDep):
    # This endpoint is async. The sync dependency ran in a thread.
    # FastAPI coordinated this automatically.
    return data
```

> "This is the same behaviour FastAPI applies to route handlers: `def` handlers run in a threadpool, `async def` handlers run in the event loop. Dependencies follow the exact same rule. You just need to choose correctly."

```
┌─────────────────────────────────────────────────────────────────┐
│                 PRACTICE CHECKPOINT 3.7                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A teammate wrote this. Is it correct? Why or why not?          │
│                                                                 │
│  async def load_feature_flags() -> dict:                        │
│      import json                                                │
│      with open("flags.json") as f:    # file I/O, BLOCKING!    │
│          return json.load(f)                                    │
│                                                                 │
│  FlagsDep = Annotated[dict, Depends(load_feature_flags)]        │
│                                                                 │
│  ─────────────────────────────────── SOLUTION ─────────────────│
│                                                                 │
│  It is WRONG. open() and json.load() are synchronous            │
│  blocking file I/O. Declaring the function async def does       │
│  NOT make them non-blocking — it just puts blocking code        │
│  directly inside the event loop, freezing it.                   │
│                                                                 │
│  Fix: change async def to def.                                  │
│                                                                 │
│  def load_feature_flags() -> dict:                              │
│      import json                                                │
│      with open("flags.json") as f:                              │
│          return json.load(f)    # Now runs in a threadpool      │
│                                                                 │
│  Alternative fix (if you want async): use aiofiles library      │
│  and await the file read properly.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.8 Decorator-Based Dependencies

**Not all dependencies inject a value. Some just enforce a rule.**

Consider an API key check. It either passes silently or raises an `HTTPException`. The endpoint never needs the key value itself — it just needs to know the check ran.

```python
from fastapi import Header, HTTPException


async def verify_api_key(x_api_key: str = Header(...)) -> None:
    """Gatekeeper: enforces API key. Returns nothing useful."""
    if x_api_key != "secret-key-123":
        raise HTTPException(
            status_code=403,
            detail="Invalid or missing API key"
        )
    # Returns None implicitly. The return value is irrelevant.
```

**Three ways to use this, from worst to best:**

```python
# ❌ MESSY: Injecting a value you don't need
# The _ prefix signals "I'm not using this" — why inject it at all?
@app.get("/admin/tasks")
async def admin_list_tasks(
    _: None = Depends(verify_api_key)
):
    return list(tasks_db.values())


# ❌ ALSO MESSY with Annotated (still injects an unused parameter):
@app.get("/admin/tasks")
async def admin_list_tasks(
    _: Annotated[None, Depends(verify_api_key)]
):
    return list(tasks_db.values())


# ✅ CLEAN: Use the dependencies= parameter on the decorator
# verify_api_key RUNS before the endpoint, but its return value
# is discarded. The function signature stays clean.
@app.get("/admin/tasks", dependencies=[Depends(verify_api_key)])
async def admin_list_tasks():
    return list(tasks_db.values())
```

**The `dependencies=` parameter accepts a list — you can stack multiple gatekeepers:**

```python
@app.delete(
    "/admin/tasks/{task_id}",
    dependencies=[
        Depends(verify_api_key),       # Must have valid API key
        Depends(check_rate_limit),     # Must not be rate limited
        Depends(log_admin_action),     # Must log this action
    ]
)
async def admin_delete_task(task_id: int):
    del tasks_db[task_id]
    # All three gatekeepers ran before we got here.
    # None of them injected anything into this function.
```

```
┌─────────────────────────────────────────────────────────────────┐
│          GATEKEEPER vs. PROVIDER DEPENDENCIES                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROVIDER (endpoint USES the value):                            │
│  ──────────────────────────────────                             │
│                                                                 │
│  async def list_tasks(                                          │
│      pagination: PaginationDep,     ← value used below          │
│  ):                                                             │
│      return tasks[pagination.skip:]  ← used here                │
│                                                                 │
│                                                                 │
│  GATEKEEPER (value NOT needed, just the side effect):           │
│  ─────────────────────────────────────────────────             │
│                                                                 │
│  @app.get("/tasks", dependencies=[Depends(verify_api_key)])     │
│  async def list_tasks():             ← clean signature          │
│      return list(tasks_db.values())  ← no pagination here       │
│                                                                 │
│                                                                 │
│  WHEN TO USE EACH:                                              │
│  ─────────────────                                              │
│  Use Annotated[T, Depends()] when the endpoint USES the         │
│  return value (pagination, current user, DB session).           │
│                                                                 │
│  Use dependencies=[Depends()] when you only need the            │
│  side effect: authentication, rate limiting, logging,           │
│  header validation, permission checking.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Back to the hotel analogy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  PROVIDER dependency = Room service                             │
│  Hotel BRINGS you something (towels, food, WiFi).               │
│  You USE it during your stay.                                   │
│                                                                 │
│  GATEKEEPER dependency = Security at the door                   │
│  Guard checks your keycard. If it's valid, you enter.           │
│  If not, you're turned away (HTTPException).                    │
│  The guard doesn't hand you anything — just enforces a rule.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 PRACTICE CHECKPOINT 3.8                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You have a check_content_type dependency that raises           │
│  HTTPException(415) if the request Content-Type header is       │
│  not "application/json". It returns nothing.                    │
│                                                                 │
│  Write the decorator for POST /tasks that:                      │
│  (a) enforces check_content_type                                │
│  (b) keeps the function signature clean (no _ parameters)       │
│  (c) still accepts a TaskCreate body                            │
│                                                                 │
│  ─────────────────────────────────── SOLUTION ─────────────────│
│                                                                 │
│  @app.post(                                                     │
│      "/tasks",                                                  │
│      status_code=201,                                           │
│      dependencies=[Depends(check_content_type)]                 │
│  )                                                              │
│  async def create_task(task: TaskCreate):                       │
│      ...                                                        │
│                                                                 │
│  The body parameter (task: TaskCreate) is still declared        │
│  normally in the function signature. dependencies=[] only       │
│  handles gatekeepers — it does not affect body parsing.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.9 Router and Application-Level Dependencies

**The `dependencies=` parameter doesn't only work on individual endpoints. It works on routers and the entire application.**

**The problem: repeating the same gatekeeper on every route.**

```python
# ❌ Copy-paste gatekeeper on every admin endpoint
@app.get("/admin/tasks",    dependencies=[Depends(verify_api_key)])
async def admin_list_tasks(): ...

@app.post("/admin/tasks",   dependencies=[Depends(verify_api_key)])
async def admin_create_task(): ...

@app.get("/admin/users",    dependencies=[Depends(verify_api_key)])
async def admin_list_users(): ...

@app.delete("/admin/tasks/{id}", dependencies=[Depends(verify_api_key)])
async def admin_delete_task(id: int): ...

# Change the auth scheme? Update EVERY line. Miss one? Silent security hole.
```

**The fix: apply the dependency to the entire router.**

```python
from fastapi import APIRouter, Depends

# All routes on this router automatically run verify_api_key
admin_router = APIRouter(
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(verify_api_key)]
)


@admin_router.get("/tasks")
async def admin_list_tasks():
    # verify_api_key ALREADY RAN before we got here.
    # No Depends() needed on the decorator or in the signature.
    return list(tasks_db.values())


@admin_router.post("/tasks", status_code=201)
async def admin_create_task(task: TaskCreate):
    # verify_api_key ran here too. Automatically.
    return create_task(task.title, task.completed)


@admin_router.delete("/tasks/{task_id}", status_code=204)
async def admin_delete_task(task_id: int):
    # And here. Always. Without any extra code.
    del tasks_db[task_id]
```

**Then register the router on the main app:**

```python
# main.py
from fastapi import FastAPI
from routers.admin import admin_router
from routers.tasks import tasks_router

app = FastAPI()
app.include_router(admin_router)
app.include_router(tasks_router)
```

**For dependencies that must apply to EVERY endpoint in the entire app:**

```python
# Applied to every single endpoint, no exceptions
app = FastAPI(
    dependencies=[Depends(log_every_request)]
)
```

**The full scope hierarchy:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DEPENDENCY SCOPE HIERARCHY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────┐      │
│  │  APPLICATION LEVEL                                    │      │
│  │  FastAPI(dependencies=[Depends(log_every_request)])   │      │
│  │  └─ Runs for EVERY endpoint in the entire app         │      │
│  │                                                       │      │
│  │  ┌─────────────────────────────────────────────┐      │      │
│  │  │  ROUTER LEVEL                               │      │      │
│  │  │  APIRouter(dependencies=[Depends(auth)])    │      │      │
│  │  │  └─ Runs for every endpoint in the router   │      │      │
│  │  │                                             │      │      │
│  │  │  ┌───────────────────────────────────┐      │      │      │
│  │  │  │  ENDPOINT LEVEL                   │      │      │      │
│  │  │  │  @router.get(...,                 │      │      │      │
│  │  │  │    dependencies=[Depends(check)]) │      │      │      │
│  │  │  │  └─ Runs for THIS endpoint only   │      │      │      │
│  │  │  │                                   │      │      │      │
│  │  │  │  ┌───────────────────────────┐    │      │      │      │
│  │  │  │  │  PARAMETER LEVEL          │    │      │      │      │
│  │  │  │  │  param: Annotated[T,      │    │      │      │      │
│  │  │  │  │    Depends(get_value)]    │    │      │      │      │
│  │  │  │  │  └─ Injects value here    │    │      │      │      │
│  │  │  │  └───────────────────────────┘    │      │      │      │
│  │  │  └───────────────────────────────────┘      │      │      │
│  │  └─────────────────────────────────────────────┘      │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                 │
│  All levels STACK. An endpoint is covered by dependencies from  │
│  all four levels simultaneously. FastAPI resolves them all.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A realistic project structure showing all levels working together:**

```python
# main.py
app = FastAPI(
    dependencies=[Depends(log_every_request)]  # Level 1: all routes
)

# Public routes — no auth
public_router = APIRouter(prefix="/public")

@public_router.get("/health")
async def health_check():
    return {"status": "ok"}  # Just log_every_request runs here


# Protected routes — API key required for all
admin_router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(verify_api_key)]   # Level 2: all admin routes
)

@admin_router.get("/stats")
async def get_stats():
    # log_every_request (app) + verify_api_key (router) both ran
    return {"total_tasks": len(tasks_db)}


@admin_router.delete(
    "/tasks/{task_id}",
    dependencies=[Depends(audit_delete_action)]  # Level 3: this endpoint only
)
async def admin_delete_task(
    task_id: int,
    pagination: PaginationDep,  # Level 4: injects a value
):
    # All four levels ran: log + verify_api_key + audit_delete + pagination
    del tasks_db[task_id]


app.include_router(public_router)
app.include_router(admin_router)
```

> "Notice how clean `admin_delete_task` is. It knows nothing about logging or API key verification — those are cross-cutting concerns handled at higher levels. It only deals with what it owns: the task deletion logic and the pagination it actually uses."

```
┌─────────────────────────────────────────────────────────────────┐
│                 PRACTICE CHECKPOINT 3.9                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You are building a multi-tenant API. You have:                 │
│  - verify_tenant: validates the X-Tenant-ID header              │
│  - log_request: logs every incoming request                     │
│  - check_subscription: verifies tenant has an active plan       │
│                                                                 │
│  log_request should run for EVERY endpoint.                     │
│  verify_tenant should run for all routes under /api.            │
│  check_subscription should run only under /api/premium.         │
│                                                                 │
│  Write the router and app setup (no endpoint bodies needed).    │
│                                                                 │
│  ─────────────────────────────────── SOLUTION ─────────────────│
│                                                                 │
│  app = FastAPI(dependencies=[Depends(log_request)])             │
│                                                                 │
│  api_router = APIRouter(                                        │
│      prefix="/api",                                             │
│      dependencies=[Depends(verify_tenant)]                      │
│  )                                                              │
│                                                                 │
│  premium_router = APIRouter(                                    │
│      prefix="/premium",                                         │
│      dependencies=[Depends(check_subscription)]                 │
│  )                                                              │
│                                                                 │
│  @premium_router.get("/reports")                                │
│  async def get_reports(): ...                                   │
│  # Runs: log_request + verify_tenant + check_subscription       │
│                                                                 │
│  api_router.include_router(premium_router)                      │
│  app.include_router(api_router)                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.10 Testing with Dependency Overrides

**One of the most important reasons to use Depends() is testability.**

When you use `Depends()`, FastAPI gives you `app.dependency_overrides` — a dictionary that swaps any dependency for any replacement during testing. The endpoint never knows the difference. It receives the same type either way.

**The scenario: testing an endpoint that uses a dependency you don't want to run in tests.**

```python
# The real dependency (imagine it connects to an external service)
async def get_pagination(
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=10, ge=1, le=100),
) -> PaginationParams:
    return PaginationParams(skip=skip, limit=limit)


PaginationDep = Annotated[PaginationParams, Depends(get_pagination)]


@app.get("/tasks")
async def list_tasks(pagination: PaginationDep):
    all_tasks = list(tasks_db.values())
    return all_tasks[pagination.skip : pagination.skip + pagination.limit]
```

**Testing with a fixed, controlled dependency:**

```python
# test_tasks.py
from fastapi.testclient import TestClient
from main import app, get_pagination, PaginationParams

client = TestClient(app)


def override_pagination_small() -> PaginationParams:
    """Always returns limit=2 for testing boundary behaviour."""
    return PaginationParams(skip=0, limit=2)


def test_list_tasks_respects_limit():
    # Install the override
    app.dependency_overrides[get_pagination] = override_pagination_small

    response = client.get("/tasks")
    assert response.status_code == 200
    assert len(response.json()) <= 2  # Our override enforces limit=2

    # CRITICAL: Always clean up!
    app.dependency_overrides.clear()
```

**How FastAPI resolves the override:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DEPENDENCY OVERRIDE RESOLUTION                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PRODUCTION (no override):                                      │
│                                                                 │
│  Request ──▶ FastAPI ──▶ get_pagination(skip=0, limit=10)       │
│                          (reads from query params, validates)   │
│                                 │                               │
│                                 ▼                               │
│                          PaginationParams(skip=0, limit=10)     │
│                                 │                               │
│                                 ▼                               │
│                          list_tasks(pagination=...)             │
│                                                                 │
│                                                                 │
│  TEST (with override):                                          │
│                                                                 │
│  Request ──▶ FastAPI ──▶ dependency_overrides lookup            │
│                          "Is get_pagination overridden? YES."   │
│                                 │                               │
│                                 ▼                               │
│                          override_pagination_small()            │
│                          (no query params, no validation)       │
│                                 │                               │
│                                 ▼                               │
│                          PaginationParams(skip=0, limit=2)      │
│                                 │                               │
│                                 ▼                               │
│                          list_tasks(pagination=...)             │
│                                                                 │
│  The endpoint receives a PaginationParams either way.           │
│  It cannot tell — and it doesn't need to.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The correct pattern: use pytest fixtures for automatic cleanup.**

```python
# conftest.py
import pytest
from fastapi.testclient import TestClient
from main import app, get_pagination, PaginationParams


@pytest.fixture
def client():
    """Client with controlled pagination (limit=2 for all tests)."""
    app.dependency_overrides[get_pagination] = lambda: PaginationParams(skip=0, limit=2)
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()  # Runs even if the test fails


@pytest.fixture
def client_unrestricted():
    """Client with no overrides — uses real dependency logic."""
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

```python
# test_tasks.py
def test_pagination_limit_enforced(client):
    response = client.get("/tasks")
    assert len(response.json()) <= 2  # Override ensures this


def test_all_tasks_returned_without_limit(client_unrestricted):
    response = client_unrestricted.get("/tasks?limit=100")
    assert response.status_code == 200
```

**A more realistic example — overriding a database session:**

```python
# This is exactly what you will do in Week 4 and Week 6:

# conftest.py
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from main import app, get_db_session  # The real DB dependency

TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"
test_engine = create_async_engine(TEST_DATABASE_URL)


@pytest.fixture
async def db_session():
    """Creates an isolated in-memory DB session for each test."""
    async with AsyncSession(test_engine) as session:
        yield session


@pytest.fixture
def client(db_session):
    """Overrides the real DB session with the test session."""
    async def override_db():
        yield db_session  # Yield dependency override!

    app.dependency_overrides[get_db_session] = override_db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

> "This fixture pattern is exactly what you'll use in Week 4 Lecture 1. Every FastAPI test suite in the industry uses `dependency_overrides` to swap database connections for test databases. The pattern you're learning now is not academic — it is production testing practice."

**The critical mistake to avoid:**

```python
# ❌ WRONG: No cleanup — override leaks into other tests!
def test_something():
    app.dependency_overrides[get_pagination] = my_override
    response = client.get("/tasks")
    assert response.status_code == 200
    # Test ends — override still installed!
    # EVERY subsequent test in the session now uses my_override silently.

# ✅ CORRECT: Always clear, even on failure
def test_something():
    app.dependency_overrides[get_pagination] = my_override
    try:
        response = client.get("/tasks")
        assert response.status_code == 200
    finally:
        app.dependency_overrides.clear()  # Guaranteed cleanup

# ✅ BEST: Use pytest fixtures (the finally is handled by yield)
# (see conftest.py above)
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 PRACTICE CHECKPOINT 3.10                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You have an endpoint that uses get_api_version (returns "v1"   │
│  or "v2" depending on the X-Api-Version header).                │
│                                                                 │
│  Write a conftest.py fixture called client_v2 that:             │
│  - Always returns "v2" from get_api_version                     │
│  - Cleans up properly after each test                           │
│                                                                 │
│  Then write a test that uses it to verify v2 behaviour.         │
│                                                                 │
│  ─────────────────────────────────── SOLUTION ─────────────────│
│                                                                 │
│  # conftest.py                                                  │
│  @pytest.fixture                                                │
│  def client_v2():                                               │
│      app.dependency_overrides[get_api_version] = lambda: "v2"  │
│      with TestClient(app) as c:                                 │
│          yield c                                                │
│      app.dependency_overrides.clear()                           │
│                                                                 │
│  # test_tasks.py                                                │
│  def test_v2_response_includes_version_key(client_v2):          │
│      response = client_v2.get("/tasks")                         │
│      assert response.status_code == 200                         │
│      assert response.json().get("version") == "v2"              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

# PART 4: LIFECYCLE & LOGGING

## 4.1 Yield Dependencies (The Room Checkout Protocol)

**Connection to Week 1, Lecture 2 (Context Managers):**

> "Remember context managers? The `with` statement that guarantees cleanup even if an error occurs? Yield dependencies are FastAPI's version of the same pattern. If you understood `with`, you'll understand this instantly."

**Side-by-side — they're the same concept:**

```python
# WEEK 1 L2: Context manager (Python builtin)
from contextlib import contextmanager

@contextmanager
def managed_file(path: str):
    file = open(path, "r")   # SETUP
    try:
        yield file            # PROVIDE to caller
    finally:
        file.close()          # CLEANUP (always runs)

# Usage:
with managed_file("data.txt") as f:
    content = f.read()
# f.close() already called, even if read() raised an error


# TODAY: Yield dependency (FastAPI's version)
async def get_db_session():
    session = create_session()    # SETUP
    try:
        yield session             # PROVIDE to endpoint
    finally:
        session.close()           # CLEANUP (always runs)

# Usage:
@app.get("/tasks")
async def list_tasks(session = Depends(get_db_session)):
    tasks = session.query(Task).all()
    return tasks
# session.close() already called, even if the endpoint raised an error
```

```
┌─────────────────────────────────────────────────────────────────┐
│           CONTEXT MANAGER vs YIELD DEPENDENCY                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Context Manager              Yield Dependency                  │
│  (Week 1 L2)                  (Today)                           │
│  ─────────────                ────────────────                  │
│                                                                 │
│  @contextmanager              async def get_thing():            │
│  def managed():                   thing = create()              │
│      thing = create()             try:                          │
│      try:                             yield thing               │
│          yield thing              finally:                      │
│      finally:                         thing.close()             │
│          thing.close()                                          │
│                                                                 │
│  with managed() as t:         @app.get("/x")                    │
│      use(t)                   async def x(t=Depends(get_thing)):│
│                                   use(t)                        │
│                                                                 │
│  SAME PATTERN:                                                  │
│  Setup → Yield → Cleanup                                        │
│                                                                 │
│  Only difference: FastAPI calls it FOR you via Depends().       │
│  You don't write the `with` statement.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The lifecycle visualized:**

```
┌─────────────────────────────────────────────────────────────────┐
│           YIELD DEPENDENCY LIFECYCLE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request arrives                                                │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────────────┐                                │
│  │ SETUP (code before yield)   │                                │
│  │ session = create_session()  │                                │
│  └──────────────┬──────────────┘                                │
│                 │                                                │
│                 ▼                                                │
│  ┌─────────────────────────────┐                                │
│  │ yield session               │                                │
│  │ ────────────────────        │                                │
│  │ Endpoint receives session   │                                │
│  │ Endpoint runs               │                                │
│  │ Response is generated       │                                │
│  └──────────────┬──────────────┘                                │
│                 │                                                │
│                 ▼ (even if endpoint raised an exception!)        │
│  ┌─────────────────────────────┐                                │
│  │ CLEANUP (code after yield)  │                                │
│  │ session.close()             │                                │
│  └──────────────┬──────────────┘                                │
│                 │                                                │
│                 ▼                                                │
│  Response sent to client                                        │
│                                                                 │
│  The HOTEL ANALOGY:                                             │
│  Setup    = Prepare the room before guest arrives               │
│  yield    = Guest uses the room                                 │
│  Cleanup  = Housekeeping cleans after checkout                  │
│  finally  = Housekeeping comes EVEN IF guest trashed the room   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A practical example — request timing:**

```python
import time


async def request_timer():
    """Yield dependency that measures endpoint execution time."""
    start = time.time()
    yield  # No value to inject — just lifecycle management
    elapsed = time.time() - start
    print(f"Request took {elapsed:.3f}s")  # We'll replace print later


@app.get("/tasks")
async def list_tasks(
    pagination: PaginationParams = Depends(),
    _timer: None = Depends(request_timer),  # _ prefix: we don't use the value
):
    all_tasks = list(tasks_db.values())
    return all_tasks[pagination.skip : pagination.skip + pagination.limit]
```

```
Request arrives → timer starts → endpoint runs → timer stops → "Request took 0.003s"
```

> "Notice the `yield` with no value. This dependency doesn't PROVIDE anything to the endpoint. It just hooks into the lifecycle: do something BEFORE, do something AFTER. Like a hotel security camera — the guest never interacts with it, but it's always running."

**CRITICAL: Always use try/finally in yield dependencies:**

```python
# ❌ WRONG: No finally — cleanup might not run on error!
async def get_db_session():
    session = create_session()
    yield session
    session.close()  # If endpoint raises, this NEVER runs!

# ✅ CORRECT: finally guarantees cleanup
async def get_db_session():
    session = create_session()
    try:
        yield session
    finally:
        session.close()  # ALWAYS runs, error or not
```

---

## 4.2 Dependency Scope and Caching

**Within a single request, FastAPI caches dependency results by default.**

```python
async def get_settings() -> dict:
    """Imagine this reads from a config file — expensive."""
    print("Loading settings...")  # How many times does this print?
    return {"max_tasks": 100, "allow_delete": True}


@app.get("/tasks")
async def list_tasks(
    settings_1: dict = Depends(get_settings),
    settings_2: dict = Depends(get_settings),  # Same dependency twice!
):
    print(settings_1 is settings_2)  # ???
    return {"tasks": []}
```

**Ask the class:**

> "How many times does `Loading settings...` print? Once or twice?"

Answer: **Once.** FastAPI sees the same dependency function used twice in the same request and caches the result.

```
Loading settings...
True
```

```
┌─────────────────────────────────────────────────────────────────┐
│              DEPENDENCY CACHING                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DEFAULT BEHAVIOR (use_cache=True):                             │
│  ──────────────────────────────────                             │
│                                                                 │
│  Request 1:                                                     │
│  ├─ Endpoint needs get_settings → Call it, cache result         │
│  ├─ Another Depends(get_settings) → Return cached result        │
│  └─ Request ends → Cache cleared                                │
│                                                                 │
│  Request 2:                                                     │
│  ├─ Endpoint needs get_settings → Call it AGAIN (new request!)  │
│  └─ Fresh result                                                │
│                                                                 │
│  SCOPE: Cache lives for ONE REQUEST only.                       │
│  Different requests ALWAYS get fresh calls.                     │
│                                                                 │
│                                                                 │
│  WHY THIS MATTERS:                                              │
│  ──────────────────                                             │
│                                                                 │
│  get_current_user might be used by:                             │
│  ├─ The endpoint itself                                         │
│  ├─ A permission-checking dependency                            │
│  └─ A logging dependency                                        │
│                                                                 │
│  Without caching: 3 database lookups for the SAME user!         │
│  With caching: 1 lookup, result shared.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**To disable caching (rare, but possible):**

```python
@app.get("/random")
async def get_random_things(
    random_1: int = Depends(get_random_number),
    random_2: int = Depends(get_random_number, use_cache=False),  # Fresh call!
):
    # random_1 and random_2 may be different
    return {"r1": random_1, "r2": random_2}
```

**The hotel analogy for caching:**

> "When you check into a hotel, the front desk creates your keycard once. If you go to the pool AND the gym AND the restaurant, they don't make a new keycard each time — they all check the same card. That's dependency caching within one visit (request)."

---

## 4.3 Why print() Will Betray You

**The problem with print for debugging APIs:**

```python
# Your endpoints in development:
@app.post("/tasks")
async def create_task(task: TaskCreate):
    print(f"Creating task: {task.title}")          # 🤷 Fine for now
    new_task = do_create_task(task)
    print(f"Created task: {new_task['id']}")        # 🤷 Fine for now
    return new_task
```

**In production, with 100 requests per second:**

```
Creating task: Buy groceries
Creating task: Write report
Created task: 47
Created task: 48
Creating task: Fix bug
ERROR: Task not found           ← Which request? Which user?
Creating task: Deploy app                What time? What endpoint?
Created task: 49                         WHICH "Task not found"?!
```

```
┌─────────────────────────────────────────────────────────────────┐
│              print() vs STRUCTURED LOGGING                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  print():                                                       │
│  ├─ No timestamp (WHEN did this happen?)                        │
│  ├─ No severity (is this info? error? critical?)                │
│  ├─ No context (which request? which user?)                     │
│  ├─ No structure (can't search, can't filter, can't aggregate)  │
│  └─ Goes to stdout only (lost when container restarts)          │
│                                                                 │
│                                                                 │
│  Structured logging:                                            │
│  ├─ Timestamp included automatically                            │
│  ├─ Log level (INFO, WARNING, ERROR, CRITICAL)                  │
│  ├─ Context bound (request_id, user_id, endpoint)               │
│  ├─ Machine-readable (JSON format, searchable)                  │
│  └─ Configurable output (console, file, log aggregator)         │
│                                                                 │
│                                                                 │
│  HOTEL ANALOGY:                                                 │
│  ──────────────                                                 │
│                                                                 │
│  print() = Yelling updates across the lobby                     │
│  structlog = Writing in the official logbook with date, time,   │
│              room number, staff name, and incident category     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 structlog Introduction

**structlog produces structured, key-value log entries:**

```python
# install: pip install structlog

import structlog

# Basic configuration for development
structlog.configure(
    processors=[
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.dev.ConsoleRenderer(),  # Pretty output for development
    ],
)

logger = structlog.get_logger()
```

**Using it:**

```python
# Instead of print:
logger.info("task_created", task_id=3, title="Buy groceries")
```

**Output (dev console):**

```
2025-02-15T10:30:00Z [info     ] task_created    task_id=3 title=Buy groceries
```

**Compare:**

```python
# ❌ print
print(f"Created task {task_id}: {title}")
# Output: Created task 3: Buy groceries
# (No timestamp, no level, no structure)

# ✅ structlog
logger.info("task_created", task_id=task_id, title=title)
# Output: 2025-02-15T10:30:00Z [info] task_created task_id=3 title=Buy groceries
# (Timestamped, leveled, structured)
```

**Log levels — when to use each:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOG LEVELS                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  logger.debug(...)    → Detailed diagnostic info                │
│                         "Looking up task_id=5 in cache"         │
│                                                                 │
│  logger.info(...)     → Normal operations                       │
│                         "task_created", "user_logged_in"        │
│                                                                 │
│  logger.warning(...)  → Something unexpected but handled        │
│                         "cache_miss", "retry_attempt"           │
│                                                                 │
│  logger.error(...)    → Something failed                        │
│                         "task_not_found", "db_connection_failed"│
│                                                                 │
│  logger.critical(...) → System is broken                        │
│                         "database_unreachable", "out_of_memory" │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Context binding — the killer feature:**

```python
import structlog

logger = structlog.get_logger()


async def get_request_logger() -> structlog.stdlib.BoundLogger:
    """Dependency that creates a logger with request context."""
    import uuid
    request_id = str(uuid.uuid4())[:8]
    # bind() returns a NEW logger with these fields on EVERY message
    return logger.bind(request_id=request_id)


@app.post("/tasks", status_code=201)
async def create_task(
    task: TaskCreate,
    log: structlog.stdlib.BoundLogger = Depends(get_request_logger),
):
    log.info("creating_task", title=task.title)

    new_task = do_create_task(task)
    log.info("task_created", task_id=new_task["id"])

    return new_task
```

**Output:**

```
2025-02-15T10:30:00Z [info] creating_task  request_id=a1b2c3d4 title=Buy groceries
2025-02-15T10:30:00Z [info] task_created   request_id=a1b2c3d4 task_id=3
```

> "Both log lines share `request_id=a1b2c3d4`. When debugging, you can search for that ID and see EVERYTHING that happened during that one request. With 100 requests per second, this is how you trace what happened."

**Using structlog in exception handlers:**

```python
logger = structlog.get_logger()


@app.exception_handler(TaskNotFoundError)
async def task_not_found_handler(request: Request, exc: TaskNotFoundError):
    logger.warning(
        "task_not_found",
        task_id=exc.task_id,
        path=str(request.url),
        method=request.method,
    )
    return JSONResponse(
        status_code=404,
        content=ErrorResponse(
            error="not_found",
            detail=str(exc),
        ).model_dump()
    )


@app.exception_handler(Exception)
async def unhandled_error_handler(request: Request, exc: Exception):
    """Catch-all: log the FULL error internally, return clean response."""
    logger.error(
        "unhandled_error",
        error_type=type(exc).__name__,
        error_message=str(exc),
        path=str(request.url),
        method=request.method,
    )
    # Client gets clean response — no stack trace!
    return JSONResponse(
        status_code=500,
        content=ErrorResponse(
            error="internal_error",
            detail="An unexpected error occurred",
        ).model_dump()
    )
```

```
┌─────────────────────────────────────────────────────────────────┐
│              THE DUAL AUDIENCE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT sees:                                                   │
│  {                                                              │
│      "error": "internal_error",                                 │
│      "detail": "An unexpected error occurred"                   │
│  }                                                              │
│  → Clean. Professional. No leaked internals.                    │
│                                                                 │
│                                                                 │
│  YOUR LOGS record:                                              │
│  {                                                              │
│      "event": "unhandled_error",                                │
│      "error_type": "KeyError",                                  │
│      "error_message": "999",                                    │
│      "path": "/tasks/999",                                      │
│      "method": "GET",                                           │
│      "timestamp": "2025-02-15T10:30:00Z",                       │
│      "level": "error"                                           │
│  }                                                              │
│  → Detailed. Searchable. Debuggable.                            │
│                                                                 │
│  THE HOTEL: Guest sees "We apologize for the inconvenience."    │
│  The incident report behind the desk has every detail.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Production tip — switch to JSON output:**

```python
# Development: pretty console output
structlog.configure(
    processors=[
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.dev.ConsoleRenderer(),  # ← Human-readable
    ],
)

# Production: JSON output (for log aggregation tools)
structlog.configure(
    processors=[
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),  # ← Machine-readable
    ],
)
```

**Production JSON output:**

```json
{"event": "task_created", "task_id": 3, "title": "Buy groceries", "timestamp": "2025-02-15T10:30:00Z", "level": "info"}
```

> "This JSON can be shipped to Elasticsearch, Datadog, CloudWatch — any log aggregation tool. You can search, filter, alert. You'll set this up properly in Week 12 (Observability). For now, just know: structlog in dev, JSON logs in prod."

---

# PART 5: COMMON MISTAKES AND BEST PRACTICES

## 5.1 Error Handling Mistakes

### Mistake 1: Returning errors as 200 OK

```python
# ❌ WRONG: Client thinks this succeeded!
@app.get("/tasks/{task_id}")
async def get_task(task_id: int):
    if task_id not in tasks_db:
        return {"error": "Task not found"}  # Status: 200 OK 😱

# ✅ CORRECT: Status code tells the truth
@app.get("/tasks/{task_id}")
async def get_task(task_id: int):
    if task_id not in tasks_db:
        raise HTTPException(status_code=404, detail="Task not found")
```

---

### Mistake 2: Leaking internal details

```python
# ❌ WRONG: Exposes database schema, library versions, file paths
@app.exception_handler(Exception)
async def bad_handler(request: Request, exc: Exception):
    import traceback
    return JSONResponse(
        status_code=500,
        content={
            "error": str(exc),
            "traceback": traceback.format_exc(),  # SECURITY RISK!
            "file": __file__,                       # SECURITY RISK!
        }
    )

# ✅ CORRECT: Clean response to client, details in logs only
@app.exception_handler(Exception)
async def good_handler(request: Request, exc: Exception):
    logger.error("unhandled_error", error=str(exc), path=str(request.url))
    return JSONResponse(
        status_code=500,
        content=ErrorResponse(
            error="internal_error",
            detail="An unexpected error occurred",
        ).model_dump()
    )
```

---

### Mistake 3: Catching too broadly in endpoints

```python
# ❌ WRONG: Swallows ALL exceptions — hides bugs!
@app.get("/tasks/{task_id}")
async def get_task(task_id: int):
    try:
        return tasks_db[task_id]
    except Exception:  # Bare except catches EVERYTHING
        return {"error": "Something went wrong"}  # And returns 200!

# ✅ CORRECT: Catch SPECIFIC exceptions, let others bubble up
@app.get("/tasks/{task_id}")
async def get_task(task_id: int):
    try:
        return tasks_db[task_id]
    except KeyError:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")
    # Any OTHER exception → bubbles up → caught by global handler
```

---

### Mistake 4: Using HTTPException in business logic

```python
# ❌ WRONG: Business logic knows about HTTP
# service.py
from fastapi import HTTPException

def get_task(task_id: int) -> dict:
    if task_id not in tasks_db:
        raise HTTPException(status_code=404, detail="Not found")
    # Now this function can ONLY be used in FastAPI endpoints.
    # A CLI tool or background job can't use it — HTTPException makes no sense there.

# ✅ CORRECT: Business logic raises domain exceptions
# service.py
def get_task(task_id: int) -> dict:
    if task_id not in tasks_db:
        raise TaskNotFoundError(task_id)  # Pure Python, no HTTP coupling
    return tasks_db[task_id]
    # Usable from endpoints, CLI tools, background jobs, tests — anywhere.
```

---

## 5.2 Dependency Injection Mistakes

### Mistake 5: Calling the dependency function directly

```python
# ❌ WRONG: Calling the function yourself — bypasses FastAPI!
@app.get("/tasks")
async def list_tasks():
    pagination = await get_pagination(skip=0, limit=10)  # Manual call
    # FastAPI doesn't know about this. No auto-docs. No injection chain.

# ✅ CORRECT: Let FastAPI call it via Depends()
@app.get("/tasks")
async def list_tasks(pagination: PaginationParams = Depends(get_pagination)):
    # FastAPI calls get_pagination, injects the result, generates docs
    ...
```

---

### Mistake 6: No cleanup in yield dependencies

```python
# ❌ WRONG: Cleanup might not run if endpoint raises!
async def get_db():
    db = create_session()
    yield db
    db.close()  # If endpoint raises, this line is SKIPPED

# ✅ CORRECT: try/finally guarantees cleanup
async def get_db():
    db = create_session()
    try:
        yield db
    finally:
        db.close()  # ALWAYS runs, error or not
```

> "This is the exact same lesson from Week 1, Lecture 2 on context managers. `try/finally` is your safety net. Always."

---

### Mistake 7: Assuming dependencies are cached across requests

```python
# This dependency is called ONCE per request (cached within that request).
# But EVERY new request calls it again. Fresh.

async def get_config():
    print("Loading config...")  # Prints on EVERY request
    return load_config()

# If loading is expensive, cache it yourself:
_cached_config = None

async def get_config():
    global _cached_config
    if _cached_config is None:
        print("Loading config (first time only)...")
        _cached_config = load_config()
    return _cached_config
```

---

## 5.3 The Decision Cheatsheet
```
┌─────────────────────────────────────────────────────────────────┐
│           WHEN TO USE WHAT                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "I want to return an error to the client"                      │
│  └─▶ raise HTTPException(status_code=..., detail=...)           │
│      (for quick, simple errors directly in the endpoint)        │
│                                                                 │
│  "I want consistent error formatting across my entire API"      │
│  └─▶ @app.exception_handler(MyException)                        │
│      (define once, applies globally)                            │
│                                                                 │
│  "I have logic repeated across many endpoints"                  │
│  └─▶ Annotated[ReturnType, Depends(my_function)]                │
│      (extract to a dependency, inject everywhere)               │
│                                                                 │
│  "I reuse the same dependency in many places"                   │
│  └─▶ MyDep = Annotated[T, Depends(fn)]  (define once,          │
│      import the alias — never repeat Depends() manually)        │
│                                                                 │
│  "I need a resource with setup AND cleanup"                     │
│  └─▶ Yield dependency with try/finally                          │
│      (like context managers but for endpoints)                  │
│                                                                 │
│  "I want complex query params grouped together"                 │
│  └─▶ Class-based dependency with Annotated[MyClass, Depends()]  │
│      (clean, typed, auto-documented)                            │
│                                                                 │
│  "My dependency calls a blocking/synchronous library"           │
│  └─▶ def (not async def) for the dependency                     │
│      (FastAPI runs it in a threadpool automatically)            │
│                                                                 │
│  "I need a gatekeeper that runs but injects nothing"            │
│  └─▶ @app.get("/x", dependencies=[Depends(check)])              │
│      (clean function signature, no unused _ parameter)          │
│                                                                 │
│  "I need auth/logging on all routes in a section"               │
│  └─▶ APIRouter(dependencies=[Depends(check)])                   │
│      (one line covers every route — no repetition)              │
│                                                                 │
│  "I need a dependency on EVERY endpoint in the app"             │
│  └─▶ FastAPI(dependencies=[Depends(check)])                     │
│      (app-level — the widest possible scope)                    │
│                                                                 │
│  "I need to test an endpoint without its real dependencies"     │
│  └─▶ app.dependency_overrides[real_dep] = fake_dep              │
│      (always clean up with app.dependency_overrides.clear())    │
│                                                                 │
│  "I need to debug what's happening in my API"                   │
│  └─▶ structlog with bound context                               │
│      (NEVER print() in production code)                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│         ERROR HANDLING & DEPENDENCIES QUICK REFERENCE           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RAISE AN HTTP ERROR:                                           │
│      raise HTTPException(status_code=404, detail="Not found")   │
│                                                                 │
│  CUSTOM EXCEPTION HANDLER:                                      │
│      @app.exception_handler(MyError)                            │
│      async def handler(request: Request, exc: MyError):         │
│          return JSONResponse(status_code=..., content={...})    │
│                                                                 │
│  FUNCTION DEPENDENCY (modern Annotated style):                  │
│      async def get_thing(param: str = Query()) -> Thing:        │
│          return Thing(param)                                    │
│                                                                 │
│      ThingDep = Annotated[Thing, Depends(get_thing)]            │
│                                                                 │
│      @app.get("/x")                                             │
│      async def endpoint(thing: ThingDep):                       │
│          use(thing)                                             │
│                                                                 │
│  CLASS DEPENDENCY (Annotated style):                            │
│      class MyParams:                                            │
│          def __init__(self, x: int = Query()):                  │
│              self.x = x                                         │
│                                                                 │
│      MyParamsDep = Annotated[MyParams, Depends()]               │
│                                                                 │
│      @app.get("/x")                                             │
│      async def endpoint(params: MyParamsDep):                   │
│          use(params.x)                                          │
│                                                                 │
│  YIELD DEPENDENCY (setup/teardown):                             │
│      async def get_resource():                                  │
│          resource = create()                                    │
│          try:                                                   │
│              yield resource                                     │
│          finally:                                               │
│              resource.close()                                   │
│                                                                 │
│  GATEKEEPER DEPENDENCY (no value injected):                     │
│      @app.get("/x", dependencies=[Depends(verify_key)])         │
│      async def endpoint(): ...                                  │
│                                                                 │
│  ROUTER-LEVEL DEPENDENCY:                                       │
│      router = APIRouter(dependencies=[Depends(verify_key)])     │
│                                                                 │
│  APP-LEVEL DEPENDENCY:                                          │
│      app = FastAPI(dependencies=[Depends(log_all)])             │
│                                                                 │
│  SYNC DEPENDENCY (blocking I/O):                                │
│      def get_config() -> dict:      ← def, NOT async def        │
│          return load_from_disk()    ← runs in threadpool        │
│                                                                 │
│  DEPENDENCY OVERRIDE IN TESTS:                                  │
│      app.dependency_overrides[real_dep] = fake_dep              │
│      # ... run tests ...                                        │
│      app.dependency_overrides.clear()  ← always clean up!      │
│                                                                 │
│  STRUCTLOG:                                                     │
│      import structlog                                           │
│      logger = structlog.get_logger()                            │
│      logger.info("event_name", key1=val1, key2=val2)            │
│      bound_log = logger.bind(request_id="abc")                  │
│                                                                 │
│  COMMON STATUS CODES:                                           │
│      200 OK            201 Created         204 No Content       │
│      400 Bad Request   401 Unauthorized    403 Forbidden        │
│      404 Not Found     409 Conflict        422 Validation Error │
│      500 Internal      502 Bad Gateway     503 Unavailable      │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ return {"error": "..."}     → Use raise HTTPException   │
│      ❌ raise HTTPException in service layer → Use domain exc   │
│      ❌ except Exception: (too broad) → Catch specific types    │
│      ❌ yield without try/finally → Always use try/finally      │
│      ❌ print() for logging → Use structlog                     │
│      ❌ async def with blocking I/O → Use def instead           │
│      ❌ Forget dependency_overrides.clear() → Leaks into tests  │
│      ❌ _ = Depends(gatekeeper) → Use dependencies=[Depends()]  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary: The Key Mental Model *(Updated)*

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ERROR HANDLING & DEPENDENCIES = PROFESSIONAL API HYGIENE       │
│                                                                 │
│  Three layers of defense:                                       │
│                                                                 │
│  ┌────────────────────────────────────────────────────┐         │
│  │  LAYER 1: Exception Handlers (Global Safety Net)   │         │
│  │  Catches domain exceptions → clean HTTP responses   │         │
│  │  Client ALWAYS gets consistent ErrorResponse shape  │         │
│  └──────────────────────┬─────────────────────────────┘         │
│                         │                                       │
│  ┌──────────────────────┴─────────────────────────────┐         │
│  │  LAYER 2: Dependencies (Shared Services)           │         │
│  │  Common logic extracted → injected into endpoints   │         │
│  │  Pagination, auth, logging — write once, use many   │         │
│  │                                                     │         │
│  │  Annotated[T, Depends(fn)] is the modern style.     │         │
│  │  Type aliases (XxxDep) eliminate repetition.        │         │
│  │  Scopes: parameter → endpoint → router → app.       │         │
│  │  Use def for sync/blocking, async def for awaitable.│         │
│  │  dependencies=[Depends()] for pure gatekeepers.     │         │
│  │  dependency_overrides[] makes it fully testable.    │         │
│  └──────────────────────┬─────────────────────────────┘         │
│                         │                                       │
│  ┌──────────────────────┴─────────────────────────────┐         │
│  │  LAYER 3: Structured Logging (Observability)       │         │
│  │  Every event recorded with context                  │         │
│  │  Debug production issues without guessing            │         │
│  └────────────────────────────────────────────────────┘         │
│                                                                 │
│  THE HOTEL ANALOGY:                                             │
│  ├─ Exception Handlers = Front desk complaint protocol          │
│  │   (Professional responses, internal details stay internal)   │
│  ├─ Annotated Dependencies = Room service order form            │
│  │   (Declare what you need — the hotel provides it, ready)     │
│  ├─ Dependency Type Aliases = Pre-printed order forms           │
│  │   (PaginationDep = standard tray — reorder without redrawing │
│  │    the menu every time)                                      │
│  ├─ Router/App Dependencies = Floor-wide house rules            │
│  │   (Every room on the admin floor requires a keycard —        │
│  │    set once at the floor level, not per-room)                │
│  ├─ Gatekeeper Dependencies = Security checkpoints              │
│  │   (Enforces entry rules — doesn't hand you anything)         │
│  ├─ Yield Dependencies = Room setup / checkout protocol         │
│  │   (Prepare before, clean up after, always)                   │
│  ├─ def Dependencies = Housekeeping in a separate office        │
│  │   (Blocking work done off the main floor so nothing stalls)  │
│  ├─ dependency_overrides = Training simulation                  │
│  │   (Swap real services for training dummies — tests run       │
│  │    without connecting to real infrastructure)                │
│  └─ structlog = Hotel logbook                                   │
│      (Everything recorded, searchable, auditable)               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Connection to Upcoming Lectures *(Updated)*

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 4, LECTURE 1 (Testing FastAPI Applications):              │
│  └─ dependency_overrides is your primary testing tool.          │
│     You'll swap real DB sessions for test DB sessions           │
│     using the exact fixture pattern from § 3.10.                │
│     pytest-asyncio + httpx.AsyncClient for async tests          │
│     (deferred from W2L2 — now finally paid off here).           │
│                                                                 │
│  WEEK 4, LECTURE 2 (API Design Principles):                     │
│  └─ APIRouter in full: include_router() customization,          │
│     nested routers, prefix/tags override at include time.       │
│     Builds directly on § 3.9.                                   │
│                                                                 │
│  WEEK 4 PROJECT: In-Memory Task Manager API                     │
│  └─ You'll USE everything from today:                           │
│     Exception hierarchy, custom handlers, Annotated Depends(),  │
│     type aliases, router-level dependencies, consistent          │
│     ErrorResponse, dependency_overrides in tests, structlog     │
│                                                                 │
│  WEEK 5 (Authentication):                                       │
│  └─ get_current_user and get_admin_user: the most important     │
│     dependency chain you'll write. CurrentUserDep type alias    │
│     will be imported into every protected router.               │
│     Chaining (§ 3.4) + router-level deps (§ 3.9) combine here. │
│                                                                 │
│  WEEK 7 (SQLAlchemy + FastAPI):                                 │
│  └─ Redis connection as a yield dependency.                     │
│     Also: model_rebuild() for cross-module circular imports     │
│     (tech debt from W3L3 — formally paid off here).             │
│                                                                 │
│  WEEK 15, LECTURE 2 (Observability):                            │
│  └─ structlog goes to production.                               │
│     JSON rendering, log aggregation, correlation IDs.           │
│     What you learned in § 4.4 is the foundation.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
