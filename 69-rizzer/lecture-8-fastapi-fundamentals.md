# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                   FASTAPI FUNDAMENTALS                          │
│                    (3.5-4 Hour Lecture)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE GAP (30 min)                                       │
│  ├─ 1.1 The Hard Way (Demonstration)                            │
│  ├─ 1.2 What is ASGI? (The Standard Plug)                       │
│  ├─ 1.3 Uvicorn: The Server Engine                              │
│  └─ 1.4 Hello, FastAPI                                          │
│                                                                 │
│  PART 2: PATH OPERATIONS (45 min)                               │
│  ├─ 2.1 Decorators Revisited (Connection to Week 1, Lecture 2)  │
│  ├─ 2.2 HTTP Methods as Decorators (Connection to W2L1)         │
│  ├─ 2.3 async def vs def in Routes (Connection to W1L3)         │
│  └─ 2.4 Route Order and Conflicts                               │
│                                                                 │
│  PART 3: EXTRACTING DATA FROM REQUESTS (60 min)                 │
│  ├─ 3.1 Path Parameters (Dynamic URLs)                          │
│  ├─ 3.2 Type Coercion Magic (Connection to Week 1, Lecture 1)   │
│  ├─ 3.3 Query Parameters (After the ?)                          │
│  ├─ 3.4 Required vs Optional (Connection to Week 1, Lecture 1)  │
│  ├─ 3.5 Combining Path and Query Parameters                     │
│  └─ 3.6 The Raw Request Object              [NEW]               │
│                                                                 │
│  PART 4: REQUEST BODY AND RESPONSES                             │
│  ├─ 4.1 When URLs Aren't Enough (The POST Problem)              │
│  ├─ 4.2 Pydantic BaseModel + Status Codes   [MODIFIED]          │
│  ├─ 4.3 Automatic Validation                                    │
│  ├─ 4.4 Combining Path + Query + Body                           │
│  ├─ 4.5 Response Models                     [NEW]               │
│  └─ 4.6 Basic Error Handling (HTTPException) [NEW]              │
│                                                                 │
│  PART 5: AUTOMATIC DOCUMENTATION                                │
│  ├─ 5.1 Swagger UI: Your API Playground (/docs)                 │
│  ├─ 5.2 ReDoc: The Readable Docs (/redoc)                       │
│  ├─ 5.3 How Type Hints Power Everything                         │
│  ├─ 5.4 The OpenAPI Schema (The Machine-Readable Contract)      │
│  └─ 5.5 Path Operation Configuration        [NEW]               │
│                                                                 │
│  PART 6: FORMS AND FILES                    [NEW]               │
│  ├─ 6.1 When JSON Isn't Enough                                  │
│  ├─ 6.2 Form Parameters (Form())                                │
│  ├─ 6.3 File Uploads (UploadFile)                               │
│  ├─ 6.4 Multiple File Uploads                                   │
│  └─ 6.5 Form + File Combined                                    │
│                                                                 │
│  PART 7: HEADERS AND COOKIES                [NEW]               │
│  ├─ 7.1 Reading Request Headers (Header())                      │
│  ├─ 7.2 Setting Response Headers                                │
│  └─ 7.3 Reading Cookies (Cookie())                              │
│                                                                 │
│  PART 8: PROJECT STRUCTURE                  [NEW]               │
│  ├─ 8.1 When One File Isn't Enough                              │
│  ├─ 8.2 APIRouter for Modular Routes                            │
│  ├─ 8.3 Including Routers (include_router)                      │
│  └─ 8.4 Typical Project Layout                                  │
│                                                                 │
│  PART 9: BEYOND JSON RESPONSES              [NEW]               │
│  ├─ 9.1 Response Classes Overview                               │
│  └─ 9.2 When to Use Each                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                   FASTAPI FUNDAMENTALS                          │
│                    (3.5-4 Hour Lecture)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE GAP (30 min)                                       │
│  ├─ 1.1 The Hard Way (Demonstration)                            │
│  ├─ 1.2 What is ASGI? (The Standard Plug)                       │
│  ├─ 1.3 Uvicorn: The Server Engine                              │
│  └─ 1.4 Hello, FastAPI                                          │
│                                                                 │
│  PART 2: PATH OPERATIONS (45 min)                               │
│  ├─ 2.1 Decorators Revisited (Connection to Week 1, Lecture 2)  │
│  ├─ 2.2 HTTP Methods as Decorators (Connection to W2L1)         │
│  ├─ 2.3 async def vs def in Routes (Connection to W1L3)         │
│  └─ 2.4 Route Order and Conflicts                               │
│                                                                 │
│  PART 3: EXTRACTING DATA FROM REQUESTS (60 min)                 │
│  ├─ 3.1 Path Parameters (Dynamic URLs)                          │
│  ├─ 3.2 Type Coercion Magic (Connection to Week 1, Lecture 1)   │
│  ├─ 3.3 Query Parameters (After the ?)                          │
│  ├─ 3.4 Required vs Optional (Connection to Week 1, Lecture 1)  │
│  └─ 3.5 Combining Path and Query Parameters                     │
│                                                                 │
│  PART 4: REQUEST BODY (45 min)                                  │
│  ├─ 4.1 When URLs Aren't Enough (The POST Problem)              │
│  ├─ 4.2 Pydantic BaseModel (Connection to W1L2 — Dataclasses)   │
│  ├─ 4.3 Automatic Validation (The Payoff of Type Hints)         │
│  └─ 4.4 Combining Path + Query + Body                           │
│                                                                 │
│  PART 5: AUTOMATIC DOCUMENTATION (30 min)                       │
│  ├─ 5.1 Swagger UI: Your API Playground (/docs)                 │
│  ├─ 5.2 ReDoc: The Readable Docs (/redoc)                       │
│  ├─ 5.3 How Type Hints Power Everything                         │
│  └─ 5.4 The OpenAPI Schema (The Machine-Readable Contract)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE GAP

## 1.1 The Hard Way

**Start with a demonstration. Show them the pain.**

> "You know HTTP. Methods, status codes, headers, JSON — you learned all of that last lecture. But knowing the protocol and writing a server that speaks it are two very different things. Let me show you."

```python
# demo_hard_way.py — A raw ASGI application. No framework. Just you and the protocol.
import json

async def app(scope, receive, send):
    """Handle HTTP requests manually. Every. Single. Detail."""
    
    if scope["type"] != "http":
        return
    
    path: str = scope["path"]
    method: str = scope["method"]
    
    # --- Manual routing (just TWO endpoints) ---
    
    if method == "GET" and path == "/":
        response_data = {"message": "Welcome to the bookstore"}
        status = 200
    
    elif method == "GET" and path.startswith("/books/"):
        # Manual path parameter extraction
        raw_id = path.split("/")[-1]
        try:
            book_id = int(raw_id)  # Manual type conversion
        except ValueError:
            response_data = {"error": f"'{raw_id}' is not a valid integer"}
            status = 422  # Manual error response
            body = json.dumps(response_data).encode("utf-8")
            await send({
                "type": "http.response.start",
                "status": status,
                "headers": [[b"content-type", b"application/json"]],
            })
            await send({"type": "http.response.body", "body": body})
            return
        
        response_data = {"book_id": book_id, "title": "Some Book"}
        status = 200
    
    else:
        response_data = {"error": "Not found"}
        status = 404
    
    # Manual JSON serialization
    body = json.dumps(response_data).encode("utf-8")
    
    # Manual response construction
    await send({
        "type": "http.response.start",
        "status": status,
        "headers": [
            [b"content-type", b"application/json"],
            [b"content-length", str(len(body)).encode()],
        ],
    })
    await send({
        "type": "http.response.body",
        "body": body,
    })
```

**Run it. Look at it. Count the lines.**

```
$ uvicorn demo_hard_way:app
INFO:     Uvicorn running on http://127.0.0.1:8000
```

It works. But look at what we had to do manually:

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHAT WE DID MANUALLY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Parse the URL path                    (path.split("/"))     │
│  2. Match HTTP methods                    (if method == "GET")  │
│  3. Extract path parameters               (path.split("/")[-1]) │
│  4. Convert types                         (int(raw_id))         │
│  5. Validate input                        (try/except)          │
│  6. Build error responses                 (manual JSON + 422)   │
│  7. Serialize to JSON                     (json.dumps)          │
│  8. Encode to bytes                       (.encode("utf-8"))    │
│  9. Set headers                           (content-type, etc.)  │
│  10. Send response in ASGI format         (two send() calls)    │
│                                                                 │
│  Lines of code: ~45                                             │
│  Endpoints: 2                                                   │
│  Documentation: None                                            │
│  Validation: Barely                                             │
│                                                                 │
│  Imagine doing this for 50 endpoints. 😱                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "We wrote 45 lines for TWO endpoints with almost no validation, no documentation, and no error handling beyond a basic type check. What happens when you need 50 endpoints? 100? With nested JSON bodies, query parameters, authentication?"

Answer: **You drown in boilerplate. You need a framework.**

**Now show the FastAPI version:**

```python
# demo_fastapi.py — Same two endpoints. Count the lines.
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Welcome to the bookstore"}

@app.get("/books/{book_id}")
async def get_book(book_id: int):
    return {"book_id": book_id, "title": "Some Book"}
```

```
$ uvicorn demo_fastapi:app
INFO:     Uvicorn running on http://127.0.0.1:8000
```

**Same behavior. 10 lines. Plus free automatic documentation, validation, and type conversion.**

```
┌─────────────────────────────────────────────────────────────────┐
│               RAW ASGI  vs  FASTAPI                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│             Raw ASGI            FastAPI                         │
│             ────────            ───────                         │
│  Lines:     ~45                 10                              │
│  Routing:   Manual if/elif     Decorators                       │
│  Params:    path.split()       Type hints                       │
│  Validation: try/except        Automatic                        │
│  Conversion: int() manually    Automatic from type hints        │
│  Errors:    Build by hand      Auto 422 with details            │
│  JSON:      json.dumps()       Automatic                        │
│  Docs:      None               /docs and /redoc for free        │
│                                                                 │
│  FastAPI didn't remove the complexity.                          │
│  It moved it behind type hints and decorators —                 │
│  things you ALREADY KNOW from Week 1.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The insight:**

> "FastAPI reads your function signature — the decorator, the parameter names, the type hints — and generates all that boilerplate FOR you. Every feature you learned in Week 1 is doing real work here. Decorators route requests. Type hints validate and convert data. Async lets you handle concurrent connections. Nothing was wasted."

---

## 1.2 What is ASGI? (The Standard Plug)

**Before we go deeper into FastAPI, understand the layer it sits on.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     WHAT IS ASGI?                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ASGI = Asynchronous Server Gateway Interface                   │
│                                                                 │
│  It's a STANDARD — a contract between two sides:                │
│                                                                 │
│  ┌─────────┐    ASGI Protocol     ┌──────────────┐              │
│  │ SERVER  │  ◀═══════════════▶   │ APPLICATION  │              │
│  │(Uvicorn)│    "We agree on      │  (FastAPI)   │              │
│  │         │     how to talk"     │              │              │
│  └─────────┘                      └──────────────┘              │
│                                                                 │
│  The server handles raw network connections.                    │
│  The application handles business logic.                        │
│  ASGI is the language they speak to each other.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Think of ASGI like USB:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE USB ANALOGY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USB lets ANY device work with ANY port.                        │
│                                                                 │
│  Mouse    ─┐                    ┌─ Laptop                       │
│  Keyboard ─┤                    ├─ Desktop                      │
│  Drive    ─┤═══ USB Standard ═══├─ Phone                        │
│  Camera   ─┘                    └─ Tablet                       │
│                                                                 │
│                                                                 │
│  ASGI lets ANY server work with ANY async Python app.           │
│                                                                 │
│  Uvicorn  ─┐                    ┌─ FastAPI                      │
│  Hypercorn─┤                    ├─ Starlette                    │
│  Daphne   ─┤═══ ASGI Standard ══├─ Django Channels              │
│  Granian  ─┘                    └─ Quart                        │
│                                                                 │
│  You can swap the server without changing your app.             │
│  You can swap the framework without changing your server.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why "Asynchronous"?**

> "Remember the event loop from Lecture 3 last week? ASGI is the ASYNC version of a web gateway. The raw ASGI app you just saw used `async def` and `await send()`. That's because ASGI is built around the async model — it can handle many connections concurrently on a single thread. This is EXACTLY what we learned in Week 1 Lecture 3."

**The full stack — where everything lives:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE LAYER CAKE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────┐              │
│  │          YOUR ROUTE FUNCTIONS                 │  ← You write │
│  │   async def get_book(book_id: int): ...       │    this      │
│  ├───────────────────────────────────────────────┤              │
│  │          FastAPI                              │  ← Routing,  │
│  │   Decorators, validation, serialization,      │    validation│
│  │   documentation generation                    │    docs      │
│  ├───────────────────────────────────────────────┤              │
│  │          Starlette                            │  ← ASGI      │
│  │   Request/Response objects, middleware,       │    toolkit   │
│  │   low-level HTTP handling                     │              │
│  ├───────────────────────────────────────────────┤              │
│  │          Uvicorn                              │  ← ASGI      │
│  │   Listens on port, manages connections,       │    server    │
│  │   translates HTTP ↔ ASGI                      │              │
│  ├───────────────────────────────────────────────┤              │
│  │          HTTP / Network                       │  ← The       │
│  │   TCP sockets, bytes on the wire              │    internet  │
│  └───────────────────────────────────────────────┘              │
│                                                                 │
│  You only touch the top layer.                                  │
│  FastAPI + Starlette handle the middle.                         │
│  Uvicorn handles the bottom.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 Uvicorn: The Server Engine

**Uvicorn is the ASGI server that runs your FastAPI application.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      UVICORN                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What it does:                                                  │
│  ├─ Listens for incoming HTTP connections                       │
│  ├─ Translates raw HTTP into ASGI events                        │
│  ├─ Passes events to your FastAPI app                           │
│  ├─ Sends FastAPI's responses back to clients                   │
│  └─ Manages the async event loop (Week 1, Lecture 3!)           │
│                                                                 │
│  What it does NOT do:                                           │
│  ├─ Routing                                                     │
│  ├─ Validation                                                  │
│  ├─ Serialization                                               │
│  └─ Any business logic                                          │
│                                                                 │
│  Uvicorn = the engine. FastAPI = the car. You = the driver.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Installing and running:**

```bash
# Install (you should already have this in your venv from Week 1, Lecture 4)
pip install fastapi uvicorn

# Run your app
uvicorn main:app --reload
#       │    │      │
#       │    │      └─ Auto-restart when code changes (dev only!)
#       │    └─ The variable name of your FastAPI instance
#       └─ The Python file (main.py)
```

**What `--reload` does:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  WITHOUT --reload:                                              │
│  ──────────────────                                             │
│  1. You edit main.py                                            │
│  2. Nothing happens. Server still runs old code.                │
│  3. You Ctrl+C, re-run uvicorn                                  │
│  4. Repeat 100 times a day. Frustrating.                        │
│                                                                 │
│  WITH --reload:                                                 │
│  ────────────────                                               │
│  1. You edit main.py                                            │
│  2. Uvicorn detects the change                                  │
│  3. Server restarts automatically                               │
│  4. You just refresh your browser                               │
│                                                                 │
│  ⚠️  --reload is for DEVELOPMENT only.                          │
│  Never use it in production (performance cost).                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 Hello, FastAPI

**Your first FastAPI application — line by line:**

```python
# main.py
from fastapi import FastAPI  # Import the framework

app = FastAPI()  # Create the application instance

@app.get("/")  # Register a route: GET requests to "/"
async def root():
    return {"message": "Hello, World"}
```

**Break down every line:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    LINE BY LINE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  from fastapi import FastAPI                                    │
│  │                                                              │
│  └─ Import the FastAPI class. This IS the framework.            │
│                                                                 │
│                                                                 │
│  app = FastAPI()                                                │
│  │                                                              │
│  ├─ Create an instance of the application                       │
│  ├─ This object holds ALL your routes, config, middleware       │
│  └─ The name "app" is convention — uvicorn looks for it         │
│     (that's what main:app means)                                │
│                                                                 │
│                                                                 │
│  @app.get("/")                                                  │
│  │                                                              │
│  ├─ @ → Decorator syntax (Week 1, Lecture 2!)                   │
│  ├─ app.get → Register this function for GET requests           │
│  ├─ "/" → The URL path this function handles                    │
│  └─ Combined: "When someone sends GET /, run this function"     │
│                                                                 │
│                                                                 │
│  async def root():                                              │
│  │                                                              │
│  ├─ async → This is a coroutine (Week 1, Lecture 3!)            │
│  ├─ def root → The function name (for your reference only)      │
│  └─ () → No parameters = no input from the request              │
│                                                                 │
│                                                                 │
│  return {"message": "Hello, World"}                             │
│  │                                                              │
│  ├─ Return a Python dict                                        │
│  └─ FastAPI automatically converts it to JSON response          │
│     (No json.dumps. No .encode. No headers. Automatic.)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Run it:**

```bash
$ uvicorn main:app --reload
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process
INFO:     Started server process
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

**Test it:**

```bash
$ curl http://127.0.0.1:8000/
{"message":"Hello, World"}
```

**Now ask the class:**

> "We returned a Python `dict`. But the client received JSON. Who did the conversion? You didn't call `json.dumps()`. You didn't set `Content-Type: application/json`. Who did?"

Answer: **FastAPI. It inspects your return value, serializes it to JSON, sets the correct headers, wraps it in an HTTP response, and sends it through ASGI. All invisible.**

---

# PART 2: PATH OPERATIONS

## 2.1 Decorators Revisited (Connection to Week 1, Lecture 2)

**Connection to what you've learned:**

> "In Week 1 Lecture 2, you learned decorators — functions that wrap other functions using the `@` syntax. You learned to read `@something` as 'pass this function into `something` and replace it.' FastAPI uses this EXACT mechanism."

```python
# Week 1, Lecture 2 — you learned this:
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("Before function call")
        result = func(*args, **kwargs)
        print("After function call")
        return result
    return wrapper

@my_decorator
def greet(name: str) -> str:
    return f"Hello, {name}"
```

```python
# FastAPI — same pattern, bigger purpose:
@app.get("/")
async def root():
    return {"message": "Hello, World"}
```

**What `@app.get("/")` actually does under the hood:**

```
┌─────────────────────────────────────────────────────────────────┐
│             WHAT THE DECORATOR DOES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @app.get("/")                                                  │
│  async def root():                                              │
│      return {"message": "Hello"}                                │
│                                                                 │
│  Is equivalent to:                                              │
│                                                                 │
│  async def root():                                              │
│      return {"message": "Hello"}                                │
│                                                                 │
│  app.get("/")(root)  ← Registers root in FastAPI's route table  │
│                                                                 │
│                                                                 │
│  What happens inside FastAPI:                                   │
│                                                                 │
│  Route Table:                                                   │
│  ┌──────────┬────────┬──────────────────────┐                   │
│  │  Method  │  Path  │  Function            │                   │
│  ├──────────┼────────┼──────────────────────┤                   │
│  │  GET     │  /     │  root()              │                   │
│  └──────────┴────────┴──────────────────────┘                   │
│                                                                 │
│  When a GET request arrives at "/", FastAPI looks up this       │
│  table and calls root().                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The insight:**

> "There's no magic. `@app.get('/')` is a decorator that registers your function in a routing table. When an HTTP request arrives, FastAPI looks up the method and path, finds the matching function, and calls it. Decorators made this syntax beautiful — but it's just a function registry."

---

## 2.2 HTTP Methods as Decorators (Connection to Week 2, Lecture 1)

**Connection to what you've learned:**

> "Last lecture you learned that HTTP methods have specific meanings: GET reads, POST creates, PUT replaces, PATCH modifies, DELETE removes. Now you'll see each method become a decorator."

```python
from fastapi import FastAPI

app = FastAPI()

# Each HTTP method from Lecture 1 has a corresponding decorator

@app.get("/books")
async def list_books():
    """GET = Read / Retrieve"""
    return [{"id": 1, "title": "Dune"}, {"id": 2, "title": "1984"}]

@app.post("/books")
async def create_book():
    """POST = Create"""
    return {"id": 3, "title": "New Book", "status": "created"}

@app.put("/books/{book_id}")
async def replace_book(book_id: int):
    """PUT = Replace entirely"""
    return {"id": book_id, "status": "replaced"}

@app.patch("/books/{book_id}")
async def update_book(book_id: int):
    """PATCH = Partial update"""
    return {"id": book_id, "status": "updated"}

@app.delete("/books/{book_id}")
async def delete_book(book_id: int):
    """DELETE = Remove"""
    return {"id": book_id, "status": "deleted"}
```

**Map it to Lecture 1's HTTP methods:**

```
┌─────────────────────────────────────────────────────────────────┐
│           HTTP METHOD → FASTAPI DECORATOR                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  W2 Lecture 1 (Theory)    │  This Lecture (Code)                │
│  ─────────────────────    │  ────────────────────               │
│  GET    → Read            │  @app.get("/path")                  │
│  POST   → Create          │  @app.post("/path")                 │
│  PUT    → Replace         │  @app.put("/path")                  │
│  PATCH  → Partial Update  │  @app.patch("/path")                │
│  DELETE → Remove          │  @app.delete("/path")               │
│                                                                 │
│  Same path, different methods = different functions:            │
│                                                                 │
│  GET  /books     → list_books()      (read all)                 │
│  POST /books     → create_book()     (create one)               │
│                                                                 │
│  The PATH identifies the RESOURCE.                              │
│  The METHOD identifies the ACTION.                              │
│  (This is REST — exactly what you learned in Lecture 1.)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The route table grows:**

```
┌──────────┬──────────────────┬──────────────────────┐
│  Method  │  Path            │  Function            │
├──────────┼──────────────────┼──────────────────────┤
│  GET     │  /books          │  list_books()        │
│  POST    │  /books          │  create_book()       │
│  PUT     │  /books/{id}     │  replace_book()      │
│  PATCH   │  /books/{id}     │  update_book()       │
│  DELETE  │  /books/{id}     │  delete_book()       │
└──────────┴──────────────────┴──────────────────────┘
```

---

## 2.3 async def vs def in Routes (Connection to Week 1, Lecture 3)

**Connection to what you've learned:**

> "In Week 1 Lecture 3, you learned that `async def` creates a coroutine — a function that can pause and let other work happen. FastAPI supports BOTH `async def` and regular `def` for routes. But they behave differently."

```python
# Both work! But they're handled differently under the hood.

# ✅ Async route — runs directly on the event loop
@app.get("/async-route")
async def async_route():
    await asyncio.sleep(1)  # Non-blocking, other requests can be served
    return {"type": "async"}

# ✅ Sync route — FastAPI runs it in a thread pool
@app.get("/sync-route")
def sync_route():
    time.sleep(1)  # Blocking, but in a separate thread, so it's OK
    return {"type": "sync"}
```

**How FastAPI handles each:**

```
┌─────────────────────────────────────────────────────────────────┐
│              async def vs def IN FASTAPI                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  async def route():                                             │
│  ──────────────────                                             │
│  FastAPI runs it DIRECTLY on the event loop.                    │
│  ├─ Use when: your function uses await (DB, HTTP calls)         │
│  ├─ NEVER use blocking calls (time.sleep, requests.get)         │
│  └─ This is the same event loop from Week 1, Lecture 3          │
│                                                                 │
│     Event Loop: ──[route A]──pause──[route B]──pause──[A]──▶    │
│                    One thread handles many requests             │
│                                                                 │
│                                                                 │
│  def route():                                                   │
│  ────────────                                                   │
│  FastAPI runs it in a SEPARATE THREAD (via thread pool).        │
│  ├─ Use when: your function does blocking I/O                   │
│  ├─ Blocking calls are OK here (they block the thread, not      │
│  │   the event loop)                                            │
│  └─ Slightly more overhead than async, but safe                 │
│                                                                 │
│     Event Loop: ──[keeps running other requests]──▶             │
│     Thread Pool: ──[route runs here, can block safely]──▶       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The critical rule:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE RULE                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ DANGEROUS: async def + blocking call                        │
│                                                                 │
│     @app.get("/bad")                                            │
│     async def bad_route():                                      │
│         time.sleep(10)  ← Freezes the ENTIRE server!            │
│         return {"oops": "blocked event loop"}                   │
│                                                                 │
│  ✅ SAFE: def + blocking call                                   │
│                                                                 │
│     @app.get("/ok")                                             │
│     def ok_route():                                             │
│         time.sleep(10)  ← Only blocks this thread               │
│         return {"fine": "thread pool handled it"}               │
│                                                                 │
│  ✅ BEST: async def + await                                     │
│                                                                 │
│     @app.get("/best")                                           │
│     async def best_route():                                     │
│         await asyncio.sleep(10)  ← Other requests keep flowing  │
│         return {"perfect": "non-blocking"}                      │
│                                                                 │
│                                                                 │
│  REMEMBER (Week 1, Lecture 3):                                  │
│  "If you call a blocking function inside async def,             │
│   your concurrency silently breaks."                            │
│  This is EXACTLY that pitfall — now in a web server context.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Route Order and Conflicts

**Routes are matched top-to-bottom. Order matters.**

```python
# ⚠️ PROBLEM: Which route handles GET /books/featured ?

@app.get("/books/{book_id}")
async def get_book(book_id: int):
    return {"book_id": book_id}

@app.get("/books/featured")
async def get_featured():
    return {"books": ["Dune", "1984"]}
```

**What happens?**

```
┌─────────────────────────────────────────────────────────────────┐
│                   ROUTE CONFLICT                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request: GET /books/featured                                   │
│                                                                 │
│  FastAPI checks routes in ORDER:                                │
│                                                                 │
│  1. /books/{book_id} → "featured" matches {book_id}? YES ✓      │
│     → Tries to convert "featured" to int → FAILS → 422 Error    │
│                                                                 │
│  2. /books/featured  → Never reached!                           │
│                                                                 │
│  The dynamic route SWALLOWED the static route.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The fix: static routes BEFORE dynamic routes:**

```python
# ✅ CORRECT ORDER: Specific before generic

@app.get("/books/featured")    # ← Static path first
async def get_featured():
    return {"books": ["Dune", "1984"]}

@app.get("/books/{book_id}")   # ← Dynamic path second
async def get_book(book_id: int):
    return {"book_id": book_id}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE RULE                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SPECIFIC routes go ABOVE generic routes.                       │
│                                                                 │
│  ✅ /books/featured          ← Exact match checked first        │
│  ✅ /books/bestsellers       ← Another exact match              │
│  ✅ /books/{book_id}         ← Catch-all checked last           │
│                                                                 │
│  Think of it like exception handling (Week 1, Lecture 2):       │
│  You catch SPECIFIC exceptions before generic ones.             │
│                                                                 │
│  except NotFoundError:  ← Specific first                        │
│  except APIError:       ← General last                          │
│                                                                 │
│  Same principle. Specific before general.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: EXTRACTING DATA FROM REQUESTS

## 3.1 Path Parameters (Dynamic URLs)

**URLs aren't always static. You need to extract data from them.**

> "In the previous lecture, you learned that REST APIs model resources with URLs. `/books/42` means 'the book with ID 42.' But how does your code get that `42`? That's what path parameters do."

```python
@app.get("/books/{book_id}")
async def get_book(book_id):
    return {"book_id": book_id, "title": "Dune"}
```

**How it works:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   PATH PARAMETERS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  URL Pattern:   /books/{book_id}                                │
│                         ─────────                               │
│                         This is a PLACEHOLDER                   │
│                                                                 │
│  Actual URL:    /books/42                                       │
│                        ──                                       │
│                        This VALUE fills the placeholder         │
│                                                                 │
│  Function:      async def get_book(book_id):                    │
│                                    ───────                      │
│                                    MUST match the placeholder   │
│                                    name exactly                 │
│                                                                 │
│  Result:        book_id = "42"  (string by default!)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The matching rule:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Path template:     /books/{book_id}/chapters/{chapter_num}     │
│  Actual request:    /books/42/chapters/7                        │
│                                                                 │
│  FastAPI extracts:                                              │
│  ├─ {book_id}      →  book_id = "42"                            │
│  └─ {chapter_num}  →  chapter_num = "7"                         │
│                                                                 │
│  The name in {braces} MUST match the function parameter name.   │
│                                                                 │
│  ❌ WRONG:                                                      │
│  @app.get("/books/{book_id}")                                   │
│  async def get_book(id):       ← "id" ≠ "book_id" → Error!      │
│                                                                 │
│  ✅ CORRECT:                                                    │
│  @app.get("/books/{book_id}")                                   │
│  async def get_book(book_id):  ← Names match ✓                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Type Coercion Magic (Connection to Week 1, Lecture 1)

**Connection to what you've learned:**

> "In Week 1 Lecture 1, you learned type hints — `int`, `str`, `float`, `bool`. You used them for documentation and mypy checking. FastAPI takes them MUCH further. It reads your type hints at RUNTIME and uses them to automatically convert and validate incoming data."

```python
# Without type hints — book_id is always a string
@app.get("/books/{book_id}")
async def get_book_untyped(book_id):
    print(type(book_id))  # <class 'str'> — always a string!
    return {"book_id": book_id}

# With type hints — book_id is automatically converted to int
@app.get("/books/{book_id}")
async def get_book_typed(book_id: int):
    print(type(book_id))  # <class 'int'> — converted automatically!
    return {"book_id": book_id}
```

**What happens under the hood:**

```
┌─────────────────────────────────────────────────────────────────┐
│               TYPE COERCION IN ACTION                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request: GET /books/42                                         │
│                                                                 │
│  1. FastAPI extracts "42" from the URL (always a string)        │
│                                                                 │
│  2. FastAPI reads the type hint: book_id: int                   │
│                                                                 │
│  3. FastAPI converts: int("42") → 42 ✅                         │
│                                                                 │
│  4. Your function receives: book_id = 42 (an actual integer)    │
│                                                                 │
│                                                                 │
│  Request: GET /books/hello                                      │
│                                                                 │
│  1. FastAPI extracts "hello" from the URL                       │
│                                                                 │
│  2. FastAPI reads the type hint: book_id: int                   │
│                                                                 │
│  3. FastAPI converts: int("hello") → FAILS ❌                   │
│                                                                 │
│  4. FastAPI returns 422 Unprocessable Entity AUTOMATICALLY:     │
│     {                                                           │
│       "detail": [{                                              │
│         "type": "int_parsing",                                  │
│         "loc": ["path", "book_id"],                             │
│         "msg": "Input should be a valid integer..."             │
│       }]                                                        │
│     }                                                           │
│                                                                 │
│  Your function is NEVER CALLED with bad data.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "Remember the raw ASGI app from Part 1? We wrote `try: book_id = int(raw_id) except ValueError:` — six lines of manual validation. FastAPI does this with three characters: `: int`. What made that possible?"

Answer: **The type hint. FastAPI reads the annotation at runtime and generates validation automatically. The type hint IS the validation.**

**All type conversions work:**

```python
@app.get("/items/{item_id}")
async def get_item(item_id: int):        # "42" → 42
    return {"item_id": item_id}

@app.get("/prices/{price}")
async def get_price(price: float):       # "19.99" → 19.99
    return {"price": price}

@app.get("/status/{is_active}")
async def get_status(is_active: bool):   # "true" → True, "1" → True
    return {"active": is_active}                  # "false" → False, "0" → False

@app.get("/users/{username}")
async def get_user(username: str):       # "alice" → "alice" (no conversion)
    return {"username": username}
```

**Visualize the type hint pipeline:**

```
┌─────────────────────────────────────────────────────────────────┐
│            THE TYPE HINT PIPELINE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  URL string ──▶ Read type hint ──▶ Convert ──▶ Validate ──▶ ✅  │
│                                                                 │
│  "/books/42"    book_id: int       int("42")   Success    Call  │
│                                    = 42        ✓          func  │
│                                                                 │
│  "/books/hi"    book_id: int       int("hi")   Fail       422   │
│                                    = ERROR     ✗          Error │
│                                                                 │
│  Week 1 Lecture 1: Type hints for documentation and mypy.       │
│  Week 2 Lecture 2: Type hints for RUNTIME validation.           │
│  Same syntax. Bigger payoff.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Query Parameters (After the ?)

**Connection to what you've learned:**

> "In Lecture 1 of this week, you learned that URLs can include query parameters after the `?` symbol: `/books?author=tolkien&limit=10`. Now let's capture them in code."

**Any function parameter that is NOT in the path template is treated as a query parameter:**

```python
@app.get("/books")
async def list_books(author: str, limit: int):
    return {
        "author": author,
        "limit": limit,
        "books": [f"Book by {author}"]
    }
```

```
Request: GET /books?author=tolkien&limit=5

FastAPI extracts:
  author = "tolkien"  (from ?author=tolkien)
  limit  = 5          (from &limit=5, converted to int)
```

**How does FastAPI know which parameters are path vs query?**

```
┌─────────────────────────────────────────────────────────────────┐
│               PATH vs QUERY — THE RULE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @app.get("/books/{book_id}")                                   │
│  async def get_book(book_id: int, verbose: bool):               │
│                     ────────      ───────────                   │
│                     IN the path   NOT in the path               │
│                     = PATH param  = QUERY param                 │
│                                                                 │
│  Request: GET /books/42?verbose=true                            │
│                      ──         ────                            │
│                      │           └─ query parameter             │
│                      └─ path parameter                          │
│                                                                 │
│                                                                 │
│  THE RULE:                                                      │
│  ─────────                                                      │
│  If the parameter name appears in {braces} in the path          │
│  → it's a PATH parameter.                                       │
│                                                                 │
│  If it does NOT appear in the path                              │
│  → it's a QUERY parameter.                                      │
│                                                                 │
│  FastAPI decides this automatically. You don't configure it.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Required vs Optional (Connection to Week 1, Lecture 1)

**Connection to what you've learned:**

> "In Week 1 Lecture 1, you learned `Optional[str]` means 'str or None.' In FastAPI, this distinction controls whether a query parameter is required or optional."

```python
from typing import Optional

@app.get("/books")
async def list_books(
    author: str,                          # Required — no default
    limit: int = 10,                      # Optional — has a default
    genre: Optional[str] = None,          # Optional — default is None
):
    results = {"author": author, "limit": limit}
    if genre:
        results["genre"] = genre
    return results
```

**What makes a parameter required vs optional:**

```
┌─────────────────────────────────────────────────────────────────┐
│             REQUIRED vs OPTIONAL                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  author: str                                                    │
│  ├─ No default value                                            │
│  ├─ REQUIRED — client MUST provide it                           │
│  └─ Missing? → 422 Error automatically                          │
│                                                                 │
│  limit: int = 10                                                │
│  ├─ Has a default value (10)                                    │
│  ├─ OPTIONAL — client CAN omit it                               │
│  └─ Missing? → Uses default (10)                                │
│                                                                 │
│  genre: Optional[str] = None                                    │
│  ├─ Type is Optional[str] (Week 1, Lecture 1!)                  │
│  ├─ Default is None                                             │
│  ├─ OPTIONAL — client CAN omit it                               │
│  └─ Missing? → Value is None                                    │
│                                                                 │
│                                                                 │
│  ⚠️  Optional[str] alone does NOT make it optional!             │
│                                                                 │
│  genre: Optional[str]           ← Still REQUIRED (no default)   │
│  genre: Optional[str] = None    ← Now OPTIONAL (has default)    │
│                                                                 │
│  The DEFAULT VALUE is what makes it optional, not the type.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Test the behavior:**

```
✅  GET /books?author=tolkien
    → {"author": "tolkien", "limit": 10}
    (limit defaults to 10, genre defaults to None)

✅  GET /books?author=tolkien&limit=5&genre=fantasy
    → {"author": "tolkien", "limit": 5, "genre": "fantasy"}
    (all provided)

❌  GET /books?limit=5
    → 422 Error: "author" field required
    (author has no default — it's mandatory)
```

---

## 3.5 Combining Path and Query Parameters

**Real endpoints use both. FastAPI handles it naturally:**

```python
@app.get("/authors/{author_id}/books")
async def get_author_books(
    author_id: int,                       # Path parameter (in {braces})
    sort_by: str = "title",               # Query parameter (not in path)
    limit: int = 20,                      # Query parameter
    include_drafts: bool = False,         # Query parameter
):
    return {
        "author_id": author_id,
        "sort_by": sort_by,
        "limit": limit,
        "include_drafts": include_drafts,
    }
```

```
Request: GET /authors/7/books?sort_by=date&limit=5&include_drafts=true
                      ─                  ────      ─               ────
                      │                  │         │               │
                      path               query     query           query
```

**Visualize the full extraction pipeline:**

```
┌─────────────────────────────────────────────────────────────────┐
│            FULL EXTRACTION PIPELINE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Incoming: GET /authors/7/books?sort_by=date&limit=5            │
│                                                                 │
│  Step 1: Match route pattern                                    │
│          /authors/{author_id}/books  ← Matches ✓                │
│                                                                 │
│  Step 2: Extract path parameters                                │
│          author_id = "7"                                        │
│                                                                 │
│  Step 3: Extract query parameters                               │
│          sort_by = "date", limit = "5"                          │
│          include_drafts → not provided → use default (False)    │
│                                                                 │
│  Step 4: Type coercion (using YOUR type hints)                  │
│          author_id: int → int("7") → 7                          │
│          sort_by: str   → "date" (already str)                  │
│          limit: int     → int("5") → 5                          │
│          include_drafts: bool → False (default)                 │
│                                                                 │
│  Step 5: Call your function with converted values               │
│          get_author_books(                                      │
│              author_id=7,                                       │
│              sort_by="date",                                    │
│              limit=5,                                           │
│              include_drafts=False                               │
│          )                                                      │
│                                                                 │
│  All of this from FUNCTION SIGNATURE alone.                     │
│  No parsing code. No conversion code. No validation code.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
## 3.6 The Raw Request Object

> "FastAPI's extraction system handles the vast majority of cases — path params, query params, and JSON bodies cover 95% of real API work. But sometimes you need to go one level deeper. The client's IP address for logging, all headers at once without naming them in advance, raw bytes in a non-standard format — for these edge cases you need the `Request` object directly."

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.get("/info")
async def get_request_info(request: Request):
    return {
        "client_ip": request.client.host,
        "method":    request.method,
        "url":       str(request.url),
        "user_agent": request.headers.get("user-agent"),
    }
```

**How does FastAPI know `request: Request` is special?**

> "Same injection system you've seen all along. FastAPI reads the TYPE annotation. When it sees `Request`, it injects the full Starlette request object — exactly as it reads `int` and knows to convert from string, it reads `Request` and knows to inject the whole object."

```
┌─────────────────────────────────────────────────────────────────┐
│            TYPE-BASED INJECTION — THE FULL PICTURE              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FastAPI reads each parameter's TYPE ANNOTATION:                │
│                                                                 │
│  book_id: int           → Extract from path or query string     │
│  book: BookCreate       → Parse from JSON request body          │
│  request: Request       → Inject the raw Starlette Request      │
│  response: Response     → Inject a mutable Response object      │
│                           (you'll use this in Part 7)           │
│                                                                 │
│  The parameter NAME doesn't matter — the TYPE does.             │
│  You can freely mix all of these in one function:               │
│                                                                 │
│  async def route(request: Request, book_id: int, limit: int=10) │
│                  ───────────────  ───────────  ──────────────   │
│                  raw object        path param   query param     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What you can access on the `Request` object:**

```
┌─────────────────────────────────────────────────────────────────┐
│              REQUEST OBJECT — USEFUL ATTRIBUTES                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  request.client.host       → Client IP address (str)            │
│  request.method            → HTTP method ("GET", "POST", ...)   │
│  request.url               → URL object (str() to convert)      │
│  request.url.path          → Just the path ("/books/42")        │
│  request.headers           → Dict-like, all request headers     │
│  request.headers.get("x")  → Read one header (None if missing)  │
│  request.query_params      → Dict-like, query string values     │
│  request.path_params       → Dict, matched path parameters      │
│  request.cookies           → Dict, all cookies from client      │
│  await request.body()      → Raw bytes of the entire body       │
│  await request.json()      → Parsed JSON body as a dict         │
│                                                                 │
│  ⚠️  If you use await request.body() or await request.json(),   │
│  FastAPI cannot also parse the body for you automatically.      │
│  Use one or the other — not both in the same endpoint.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When to reach for `Request` — and when not to:**

```
┌─────────────────────────────────────────────────────────────────┐
│              REQUEST OBJECT — USE DECISION                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ USE Request WHEN:                                           │
│  ├─ You need the client's IP (logging, rate limiting, audit)    │
│  ├─ You need ALL headers without knowing names in advance       │
│  ├─ You need middleware-like behaviour inside a single route    │
│  └─ You need raw bytes in a non-JSON, non-form binary format    │
│                                                                 │
│  ❌ DON'T use Request WHEN FastAPI already covers it:           │
│                                                                 │
│  ❌ request.query_params["limit"]   →  limit: int = 10          │
│  ❌ request.headers["user-agent"]   →  Header() — see Part 7    │
│  ❌ request.path_params["book_id"]  →  /books/{book_id}         │
│  ❌ await request.json()            →  body: MyModel            │
│                                                                 │
│  Request is the escape hatch. FastAPI's extraction system is    │
│  cleaner, type-safe, and auto-documented. Use it first.         │
│  Reach for Request only when extraction can't cover your case.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   PRACTICE CHECKPOINT                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Write a GET /debug endpoint that returns a JSON response with: │
│    - The client's IP address                                    │
│    - The value of the "Accept" header (None if missing)         │
│    - The full URL as a string                                   │
│    - A query parameter verbose (bool, default False)            │
│                                                                 │
│  Challenge: Can you use both Request and a regular query        │
│  parameter in the same function signature? If so, where must    │
│  Request appear relative to the defaulted parameter?            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solution:**

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.get("/debug")
async def debug_info(request: Request, verbose: bool = False):
    #               ───────────────── ─────────────────────
    #               Request BEFORE    Regular query param.
    #               the defaulted     Works exactly as
    #               param — required  normal.
    #               by Python syntax.
    return {
        "client_ip": request.client.host,
        "accept":    request.headers.get("accept"),
        "url":       str(request.url),
        "verbose":   verbose,
    }
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOLUTION EXPLAINED                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Yes — Request and regular parameters coexist freely.           │
│                                                                 │
│  The ordering rule:                                             │
│  request: Request       → no default value (required in Python) │
│  verbose: bool = False  → has a default value                   │
│                                                                 │
│  Python requires non-defaulted arguments BEFORE defaulted       │
│  arguments. Request must come first.                            │
│                                                                 │
│  FastAPI injects Request by type. It extracts verbose from      │
│  the query string (?verbose=true). They do not interfere.       │
│                                                                 │
│  GET /debug?verbose=true                                        │
│  → {"client_ip": "127.0.0.1", ..., "verbose": true}            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: REQUEST BODY

## 4.1 When URLs Aren't Enough (The POST Problem)

**Ask the class:**

> "You want to create a new book. The book has a title, author, price, publication year, and a list of genres. Can you put all that in a URL?"

```
GET /books?title=The%20Lord%20of%20the%20Rings&author=Tolkien&price=29.99
    &year=1954&genres=fantasy&genres=adventure&isbn=978-0618640157
    &publisher=Houghton%20Mifflin&pages=1178&...
```

**Problems:**

```
┌─────────────────────────────────────────────────────────────────┐
│           WHY URLS AREN'T ENOUGH                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. URL LENGTH LIMITS                                           │
│     Browsers and servers limit URL length (~2000-8000 chars).   │
│     Complex data overflows.                                     │
│                                                                 │
│  2. NO STRUCTURE                                                │
│     Query params are flat key=value pairs.                      │
│     What about nested data? Lists of objects?                   │
│                                                                 │
│  3. WRONG SEMANTICS                                             │
│     Creating data = POST (Week 2, Lecture 1).                   │
│     POST sends data in the BODY, not the URL.                   │
│                                                                 │
│  4. SECURITY                                                    │
│     URLs appear in server logs, browser history, referer        │
│     headers. Sensitive data should not be in the URL.           │
│                                                                 │
│                                                                 │
│  SOLUTION: Send structured data in the REQUEST BODY as JSON.    │
│                                                                 │
│  POST /books                                                    │
│  Content-Type: application/json                                 │
│  {                                                              │
│      "title": "The Lord of the Rings",                          │
│      "author": "Tolkien",                                       │
│      "price": 29.99,                                            │
│      "year": 1954,                                              │
│      "genres": ["fantasy", "adventure"]                         │
│  }                                                              │
│                                                                 │
│  Structured. Unlimited size. Correct semantics. Secure.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Pydantic BaseModel (Connection to Week 1, Lecture 2 — Dataclasses)

**Connection to what you've learned:**

> "In Week 1 Lecture 2, you learned dataclasses — a clean way to define data structures with typed fields. Pydantic's `BaseModel` is like a dataclass that VALIDATES data automatically. The shape is almost identical."

```python
# Week 1, Lecture 2 — Dataclass (you know this)
from dataclasses import dataclass

@dataclass
class Book:
    title: str
    author: str
    price: float
    year: int
```

```python
# Week 2, Lecture 2 — Pydantic BaseModel (new)
from pydantic import BaseModel

class Book(BaseModel):
    title: str
    author: str
    price: float
    year: int
```

**Side by side:**

```
┌─────────────────────────────────────────────────────────────────┐
│           DATACLASS vs PYDANTIC BASEMODEL                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Dataclass (Week 1)            │  BaseModel (This lecture)      │
│  ──────────────────            │  ─────────────────────         │
│  @dataclass                    │  class Book(BaseModel):        │
│  class Book:                   │      title: str                │
│      title: str                │      author: str               │
│      author: str               │      price: float              │
│      price: float              │      year: int                 │
│      year: int                 │                                │
│                                │                                │
│  ✅ Typed fields               │  ✅ Typed fields                │
│  ✅ Auto __init__              │  ✅ Auto __init__               │
│  ✅ Auto __repr__              │  ✅ Auto __repr__               │
│  ❌ No validation              │  ✅ AUTOMATIC VALIDATION        │
│  ❌ No JSON parsing            │  ✅ AUTOMATIC JSON PARSING      │
│  ❌ No serialization           │  ✅ AUTOMATIC SERIALIZATION     │
│  ❌ No coercion                │  ✅ AUTOMATIC TYPE COERCION     │
│                                │                                │
│  Dataclass says: "These are my fields."                         │
│  BaseModel says: "These are my fields, AND I'll enforce them."  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
> "We've been writing `@app.get()` and `@app.post()` without ever setting the HTTP status code. By default FastAPI returns `200 OK` for everything — including POST, which should return `201 Created`. Status codes are not decoration. Clients and API consumers use them to decide what happened and how to react."

```
┌─────────────────────────────────────────────────────────────────┐
│            STATUS CODES IN ROUTE DECORATORS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Every route decorator accepts a status_code parameter:         │
│                                                                 │
│  @app.get("/books",         status_code=200)  ← default (ok)    │
│  @app.post("/books",        status_code=201)  ← 201 Created     │
│  @app.put("/books/{id}",    status_code=200)  ← default (ok)    │
│  @app.patch("/books/{id}",  status_code=200)  ← default (ok)    │
│  @app.delete("/books/{id}", status_code=204)  ← 204 No Content  │
│                                                                 │
│  The default is 200 for ALL methods. That is correct for GET    │
│  but WRONG for POST (should be 201) and DELETE (often 204).     │
│                                                                 │
│  Convention (from RFC 9110):                                    │
│  ├─ 200 OK           → Successful read or full update           │
│  ├─ 201 Created      → Resource was successfully created        │
│  ├─ 204 No Content   → Success, nothing to return               │
│  ├─ 400 Bad Request  → Syntactically valid but logically wrong  │
│  ├─ 404 Not Found    → Resource does not exist                  │
│  └─ 422 Unprocessable → Validation failed (FastAPI auto)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Using the `fastapi.status` constants (cleaner alternative to raw integers):**

```python
from fastapi import FastAPI, status

app = FastAPI()

@app.post("/books",        status_code=status.HTTP_201_CREATED)
@app.delete("/books/{id}", status_code=status.HTTP_204_NO_CONTENT)
@app.get("/books",         status_code=status.HTTP_200_OK)  # redundant but explicit
```

```
┌─────────────────────────────────────────────────────────────────┐
│              INTEGER vs STATUS CONSTANTS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  status_code=201                                                │
│  └─ Works. But what is 201? You have to know by memory.         │
│                                                                 │
│  status_code=status.HTTP_201_CREATED                            │
│  └─ Self-documenting. Readable. IDE autocomplete. Typo-safe.    │
│                                                                 │
│  Both produce identical HTTP responses.                         │
│  This course uses integer literals for brevity.                 │
│  Standardise whichever your team prefers, but be consistent.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Special case — 204 No Content:**

```python
@app.delete("/books/{book_id}", status_code=204)
async def delete_book(book_id: int):
    # With status_code=204, FastAPI sends NO body at all.
    # Your return value is completely IGNORED.
    return None  # ← FastAPI will NOT serialise or send this
```

```
┌─────────────────────────────────────────────────────────────────┐
│                204 — THE SILENT SUCCESS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP 204 means "success, but I have nothing to tell you."      │
│                                                                 │
│  When you set status_code=204:                                  │
│  ├─ FastAPI ignores your return value entirely                  │
│  ├─ The response body is EMPTY — not even {}                    │
│  └─ Content-Length is 0 or absent                               │
│                                                                 │
│  This is correct for DELETE. The resource is gone.              │
│  Returning {"status": "deleted"} after a DELETE is technically  │
│  valid (use 200 then) but 204 is the idiomatic REST choice.     │
│                                                                 │
│  Pick one. Be consistent across your whole API.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Updated code from 4.2 — POST decorator now correct:**

```python
# ✅ Updated: status_code=201 on all POST endpoints
@app.post("/books", status_code=201)
async def create_book(book: BookCreate):
    return {
        "message": "Book created",
        "book": {
            "title":  book.title,
            "author": book.author,
            "price":  book.price,
            "year":   book.year,
        }
    }
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   PRACTICE CHECKPOINT                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Without looking at the table above, answer these:              │
│                                                                 │
│  1. What status_code should POST /tasks use?                    │
│  2. What status_code should DELETE /tasks/{id} use if you want  │
│     to return nothing in the body?                              │
│  3. If DELETE returns 204 and your function returns a dict,     │
│     what does the client actually receive?                      │
│  4. Write three decorators — one each for retrieve, create,     │
│     and delete a task. Include only the decorator lines.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solution:**

```python
@app.get("/tasks/{task_id}")              # 200 is default — can omit
async def get_task(task_id: int): ...

@app.post("/tasks", status_code=201)      # MUST specify — default is wrong
async def create_task(): ...

@app.delete("/tasks/{task_id}", status_code=204)  # Body will be empty
async def delete_task(task_id: int):
    return None   # FastAPI ignores this — response body is always empty
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOLUTION EXPLAINED                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GET  → 200. Omitting status_code is conventional and fine.     │
│                                                                 │
│  POST → 201. MUST specify. FastAPI defaults POST to 200 too.    │
│  Without status_code=201 the client gets the wrong signal.      │
│                                                                 │
│  DELETE + 204: the client receives an empty body — nothing,     │
│  not even a null or {}. return None is meaningless code here    │
│  but harmless; FastAPI strips it.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Use BaseModel as a request body:**

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class BookCreate(BaseModel):
    title: str
    author: str
    price: float
    year: int

@app.post("/books")
async def create_book(book: BookCreate):
    # book is already validated and typed!
    # book.title is a str, book.price is a float, etc.
    return {
        "message": "Book created",
        "book": {
            "title": book.title,
            "author": book.author,
            "price": book.price,
            "year": book.year,
        }
    }
```

**How does FastAPI know `book` is a request body?**

```
┌─────────────────────────────────────────────────────────────────┐
│           HOW FASTAPI DECIDES PARAMETER SOURCE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FastAPI checks the TYPE ANNOTATION to decide:                  │
│                                                                 │
│  book_id: int         → Simple type → Path or Query parameter   │
│  book: BookCreate     → Pydantic model → REQUEST BODY           │
│                                                                 │
│                                                                 │
│  Full decision tree:                                            │
│                                                                 │
│  Is the parameter name in the path template {braces}?           │
│  ├─ YES → Path parameter                                        │
│  └─ NO  → Is the type a Pydantic BaseModel?                     │
│           ├─ YES → Request body (parsed from JSON)              │
│           └─ NO  → Query parameter                              │
│                                                                 │
│  That's it. No configuration. No decorators.                    │
│  The type hint tells FastAPI everything.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Test it with curl:**

```bash
$ curl -X POST http://127.0.0.1:8000/books \
    -H "Content-Type: application/json" \
    -d '{"title": "Dune", "author": "Herbert", "price": 15.99, "year": 1965}'

{"message":"Book created","book":{"title":"Dune","author":"Herbert",
"price":15.99,"year":1965}}
```

---

## 4.3 Automatic Validation (The Payoff of Type Hints)

**What happens when the client sends bad data?**

```bash
# Missing required field
$ curl -X POST http://127.0.0.1:8000/books \
    -H "Content-Type: application/json" \
    -d '{"title": "Dune", "author": "Herbert"}'
```

**FastAPI responds automatically:**

```json
{
  "detail": [
    {
      "type": "missing",
      "loc": ["body", "price"],
      "msg": "Field required",
      "input": {"title": "Dune", "author": "Herbert"}
    },
    {
      "type": "missing",
      "loc": ["body", "year"],
      "msg": "Field required",
      "input": {"title": "Dune", "author": "Herbert"}
    }
  ]
}
```

**Status code: 422 Unprocessable Entity**

```
┌─────────────────────────────────────────────────────────────────┐
│           AUTOMATIC VALIDATION CASES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Missing field:                                                 │
│  ├─ Input: {"title": "Dune"}                                    │
│  ├─ Error: "author", "price", "year" are required               │
│  └─ Status: 422                                                 │
│                                                                 │
│  Wrong type:                                                    │
│  ├─ Input: {"title": "Dune", "author": "X", "price": "free",    │
│  │          "year": 1965}                                       │
│  ├─ Error: "price" — input should be a valid number             │
│  └─ Status: 422                                                 │
│                                                                 │
│  Wrong JSON:                                                    │
│  ├─ Input: {not valid json                                      │
│  ├─ Error: JSON decode error                                    │
│  └─ Status: 422                                                 │
│                                                                 │
│  No body at all:                                                │
│  ├─ Input: (empty)                                              │
│  ├─ Error: "body" — field required                              │
│  └─ Status: 422                                                 │
│                                                                 │
│                                                                 │
│  Your function is NEVER called with invalid data.               │
│  By the time your code runs, everything is validated,           │
│  converted, and typed correctly.                                │
│                                                                 │
│  You wrote ZERO validation code. The BaseModel IS the           │
│  validation. The type hints ARE the rules.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Optional fields in the body work the same way:**

```python
from typing import Optional

class BookCreate(BaseModel):
    title: str                            # Required
    author: str                           # Required
    price: float                          # Required
    year: int                             # Required
    genre: Optional[str] = None           # Optional, defaults to None
    pages: int = 0                        # Optional, defaults to 0
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Same pattern as query parameters:                              │
│                                                                 │
│  No default → Required                                          │
│  Has default → Optional                                         │
│                                                                 │
│  Type hints drive everything. Consistently. Everywhere.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Combining Path + Query + Body

**Real endpoints often use all three at once:**

```python
class BookUpdate(BaseModel):
    title: Optional[str] = None
    price: Optional[float] = None

@app.put("/authors/{author_id}/books/{book_id}")
async def update_book(
    author_id: int,                  # Path parameter  (in {braces})
    book_id: int,                    # Path parameter  (in {braces})
    book: BookUpdate,                # Request body    (Pydantic model)
    notify_author: bool = False,     # Query parameter (simple type, not in path)
):
    return {
        "author_id": author_id,
        "book_id": book_id,
        "updates": book.model_dump(exclude_unset=True),
        "notify_author": notify_author,
    }
```

```
Request:

  PUT /authors/7/books/42?notify_author=true
  Content-Type: application/json
  {"title": "Dune: Revised Edition", "price": 18.99}
       ─             ──               ────            ─────
       │             │                │               │
       path          path             body            query
```

**Visualize the complete extraction:**

```
┌─────────────────────────────────────────────────────────────────┐
│           THE COMPLETE PICTURE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PUT /authors/7/books/42?notify_author=true                     │
│  Body: {"title": "Dune: Revised", "price": 18.99}               │
│                                                                 │
│  ┌─────────────────┐                                            │
│  │  URL PATH        │──▶ author_id: int = 7                     │
│  │  /authors/7/...  │    book_id: int = 42                      │
│  ├─────────────────┤                                            │
│  │  QUERY STRING    │──▶ notify_author: bool = True             │
│  │  ?notify_author  │                                           │
│  ├─────────────────┤                                            │
│  │  REQUEST BODY    │──▶ book: BookUpdate(                      │
│  │  {"title":...}   │       title="Dune: Revised",              │
│  │                  │       price=18.99                         │
│  └─────────────────┘   )                                        │
│                                                                 │
│                                                                 │
│  THREE data sources. ONE function signature.                    │
│  FastAPI routes each parameter to the right source              │
│  based on the SAME rules you already learned:                   │
│                                                                 │
│  In {braces}?     → Path                                        │
│  Pydantic model?  → Body                                        │
│  Everything else? → Query                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
## 4.5 Response Models

> "You've been returning dicts and FastAPI serialises whatever is in them. But consider: what if that dict contains an `internal_notes` field meant only for backend logic? Or a `hashed_password` that should never leave the server? Right now FastAPI would send both to the client without hesitation. The `response_model` parameter is a whitelist for your output — only the fields you declare reach the client."

**The problem:**

```python
class BookCreate(BaseModel):
    title: str
    author: str
    price: float
    internal_notes: str = ""    # Backend tracking field — must NEVER be exposed

@app.post("/books", status_code=201)
async def create_book(book: BookCreate):
    return {
        "id": 1,
        "title": book.title,
        "author": book.author,
        "price": book.price,
        "internal_notes": "Purchased via reseller at 40% margin",  # 😱 EXPOSED
    }
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE LEAKAGE PROBLEM                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Without response_model, FastAPI returns EVERYTHING in the      │
│  dict or model your function returns — including:               │
│                                                                 │
│  ├─ Internal fields you forgot to remove                        │
│  ├─ Sensitive data: hashed_password, cost_price, is_admin       │
│  ├─ Database row internals (raw ORM attributes)                 │
│  └─ Fields computed only for internal business logic            │
│                                                                 │
│  You are trusting yourself to manually exclude every field      │
│  every time. In a real codebase, someone will forget.           │
│  It is a data breach waiting to happen.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The solution — separate input and output models:**

```python
from pydantic import BaseModel
from typing import Optional

# Input model: what the CLIENT sends you
class BookCreate(BaseModel):
    title: str
    author: str
    price: float
    year: int
    internal_notes: str = ""       # Accepted on input — stays server-side

# Output model: what YOUR API returns to the client
class BookResponse(BaseModel):
    id: int
    title: str
    author: str
    year: int
    # price is intentionally absent (business decision)
    # internal_notes is intentionally absent — never expose this

@app.post("/books", status_code=201, response_model=BookResponse)
async def create_book(book: BookCreate):
    # Return a dict with MORE fields than BookResponse declares.
    # FastAPI filters it down to ONLY what BookResponse allows.
    return {
        "id": 1,
        "title": book.title,
        "author": book.author,
        "year": book.year,
        "price": book.price,              # ← FILTERED OUT (not in BookResponse)
        "internal_notes": book.internal_notes,  # ← FILTERED OUT
    }
```

**What happens under the hood:**

```
┌─────────────────────────────────────────────────────────────────┐
│             RESPONSE MODEL FILTERING PIPELINE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your function returns:                                         │
│  { "id":1, "title":"Dune", "author":"Herbert",                  │
│    "year":1965, "price":15.99, "internal_notes":"..." }         │
│                                                                 │
│  FastAPI runs response_model=BookResponse on it:                │
│                                                                 │
│  BookResponse declares:  id, title, author, year                │
│  Your dict has:          id ✓  title ✓  author ✓  year ✓        │
│                          price ✗   internal_notes ✗             │
│                                                                 │
│  Client receives:                                               │
│  { "id":1, "title":"Dune", "author":"Herbert", "year":1965 }    │
│                                                                 │
│  price and internal_notes were STRIPPED by FastAPI.             │
│  You returned them. The client never saw them.                  │
│  response_model = a whitelist for your output.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Also documented in `/docs` automatically:**

```
┌─────────────────────────────────────────────────────────────────┐
│           RESPONSE MODEL IN SWAGGER UI                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Without response_model:                                        │
│  └─ Swagger shows "Successful Response" with no schema          │
│                                                                 │
│  With response_model=BookResponse:                              │
│  └─ Swagger shows the EXACT JSON shape clients will receive     │
│     ├─ Field names and types                                    │
│     ├─ Which fields are required vs optional                    │
│     └─ Example values generated from the schema                 │
│                                                                 │
│  response_model is not just a filter — it IS the documentation  │
│  for your response. The output contract is in your code.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**`response_model_exclude_unset` for PATCH endpoints:**

> "PATCH sends only the fields the client wants to change. Without `response_model_exclude_unset`, the response echoes `null` for every field the client never touched. That's noisy and confusing."

```python
class BookUpdate(BaseModel):
    title:  Optional[str]   = None
    price:  Optional[float] = None
    year:   Optional[int]   = None

@app.patch(
    "/books/{book_id}",
    response_model=BookUpdate,
    response_model_exclude_unset=True,
)
async def update_book(book_id: int, book: BookUpdate):
    # Client sends: {"price": 12.99}
    # WITHOUT exclude_unset → {"title": null, "price": 12.99, "year": null}
    # WITH    exclude_unset → {"price": 12.99}    ← only what was sent
    return book
```

**The core mental model:**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE TWO-MODEL PATTERN                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  For every resource, maintain TWO Pydantic models:              │
│                                                                 │
│  BookCreate (input)          BookResponse (output)              │
│  ───────────────────         ────────────────────               │
│  title: str                  id: int                            │
│  author: str                 title: str                         │
│  price: float                author: str                        │
│  year: int                   year: int                          │
│  internal_notes: str         (everything else stripped)         │
│                                                                 │
│  POST /books                                                    │
│  ├─ Reads  body   as BookCreate    ← what the client sends      │
│  └─ Writes output as BookResponse  ← what the client gets back  │
│                                                                 │
│  This two-model pattern appears in every professional API.      │
│  Week 3, Lecture 3 (Pydantic Deep Dive) goes deeper into        │
│  response_model_include/exclude sets and full serialisation      │
│  control.                                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   PRACTICE CHECKPOINT                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You are building a user registration endpoint.                 │
│                                                                 │
│  Input accepts:   email, password, username                     │
│  Response shows:  id, email, username, last_login               │
│  Response NEVER shows: password                                 │
│                                                                 │
│  1. Define UserCreate and UserResponse Pydantic models.         │
│     last_login should be optional with a default of None.       │
│  2. Write POST /users with the correct status_code and          │
│     response_model.                                             │
│  3. Return a fake dict that INCLUDES password — confirm that    │
│     FastAPI strips it.                                          │
│  4. Your fake dict does NOT include last_login. What does       │
│     FastAPI send back for that field?                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solution:**

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class UserCreate(BaseModel):
    email: str
    password: str        # Accepted on input
    username: str

class UserResponse(BaseModel):
    id: int
    email: str
    username: str
    last_login: Optional[str] = None   # Optional with default

@app.post("/users", status_code=201, response_model=UserResponse)
async def create_user(user: UserCreate):
    return {
        "id": 42,
        "email":    user.email,
        "username": user.username,
        "password": user.password,   # ← In the dict, STRIPPED by response_model
        # "last_login" is absent from this dict
    }
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOLUTION EXPLAINED                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  password:   In the return dict, NOT in UserResponse.           │
│  FastAPI strips it. Client never sees it. ✅                     │
│                                                                 │
│  last_login: In UserResponse but NOT in the return dict.        │
│  FastAPI uses UserResponse's DEFAULT (None). The client gets    │
│  "last_login": null. No error.                                  │
│                                                                 │
│  If last_login had no default (last_login: str — required),     │
│  FastAPI would raise an internal server error because the       │
│  return data doesn't satisfy the declared response model.       │
│  Always give response model fields a default if they may be     │
│  absent from your return value.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
## 4.6 Basic Error Handling (HTTPException)

> "Every route so far either succeeds or triggers a 422 validation error automatically. But real APIs have a third outcome: a business logic failure. `book_id=99` is a perfectly valid integer — it passes type validation — but no book with that ID exists. FastAPI's automatic 422 system won't fire here. You have to tell the client '404 Not Found' yourself."

**The two types of errors in FastAPI:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TWO TYPES OF API ERRORS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TYPE 1: Validation Error → 422 Unprocessable Entity           │
│  ─────────────────────────────────────────────────             │
│  Happens when: Wrong type, missing required field, bad format   │
│  Who handles it: FastAPI AUTOMATICALLY                          │
│  You write: nothing. It's free.                                 │
│  Example: GET /books/hello → "hello" can't convert to int       │
│                                                                 │
│                                                                 │
│  TYPE 2: Business Logic Error → 4xx (your choice)              │
│  ─────────────────────────────────────────────────             │
│  Happens when: Data is valid, but the result is wrong           │
│    - book_id=99 is a valid int — but book 99 doesn't exist      │
│    - User tries to delete someone else's resource               │
│    - Duplicate email on registration                            │
│  Who handles it: YOU, with HTTPException                        │
│  You write: raise HTTPException(status_code=..., detail=...)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**`HTTPException` in action:**

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

# A fake in-memory "database"
fake_books_db: dict[int, dict] = {
    1: {"id": 1, "title": "Dune",   "author": "Herbert"},
    2: {"id": 2, "title": "1984",   "author": "Orwell"},
}

@app.get("/books/{book_id}")
async def get_book(book_id: int):
    if book_id not in fake_books_db:
        raise HTTPException(
            status_code=404,
            detail=f"Book with id {book_id} not found"
        )
    return fake_books_db[book_id]   # Only reached if the book exists
```

**What the client receives on a 404:**

```bash
$ curl http://127.0.0.1:8000/books/99

HTTP/1.1 404 Not Found
content-type: application/json

{"detail": "Book with id 99 not found"}
```

```
┌─────────────────────────────────────────────────────────────────┐
│              HTTPException — HOW IT WORKS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  raise HTTPException(status_code=404, detail="Not found")       │
│                                                                 │
│  ├─ status_code: any valid HTTP status code (4xx or 5xx)        │
│  ├─ detail: a string, dict, or list — goes in the response body │
│  │          {"detail": YOUR_VALUE_HERE}                         │
│  └─ This is a Python exception — use raise, NOT return          │
│                                                                 │
│                                                                 │
│  ❌ VERY COMMON MISTAKE:                                        │
│                                                                 │
│  return HTTPException(status_code=404, detail="...")            │
│  ↑ This RETURNS the exception object as a JSON blob.            │
│    The HTTP status is still 200. The client is confused.        │
│                                                                 │
│  ✅ CORRECT:                                                    │
│                                                                 │
│  raise HTTPException(status_code=404, detail="...")             │
│  ↑ FastAPI catches this and sends a real 404 response.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Common status codes for business logic errors:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHICH STATUS CODE TO USE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  404 Not Found                                                  │
│  └─ The resource doesn't exist at all                           │
│     GET /books/99 → no book 99 in the system                    │
│                                                                 │
│  400 Bad Request                                                │
│  └─ Input is structurally valid but logically wrong             │
│     "Start date cannot be after end date"                       │
│     "Quantity cannot be negative"                               │
│                                                                 │
│  409 Conflict                                                   │
│  └─ The action conflicts with the current state                 │
│     Registering an email that already exists                    │
│     Creating a resource that would create a duplicate           │
│                                                                 │
│  403 Forbidden                                                  │
│  └─ Authenticated but not authorised for this action            │
│     Deleting someone else's resource                            │
│                                                                 │
│  401 Unauthorized                                               │
│  └─ Not authenticated at all (no token, expired token)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**`detail` can be a structured dict for clients that need to parse errors:**

```python
@app.post("/users", status_code=201)
async def create_user(user: UserCreate):
    if user_exists_in_db(user.email):
        raise HTTPException(
            status_code=409,
            detail={
                "error":   "email_conflict",
                "message": f"An account with {user.email} already exists",
                "field":   "email",
            }
        )
    return {"id": 1, "email": user.email}
```

> "String detail for simple cases. Dict detail when clients need to programmatically parse the error — for example, to highlight the specific form field that caused the problem. Pick a format and enforce it consistently across your whole API."

```
┌─────────────────────────────────────────────────────────────────┐
│                   PRACTICE CHECKPOINT                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Using this fake database:                                      │
│    fake_tasks = {                                               │
│        1: {"id": 1, "title": "Buy milk", "done": False},        │
│        2: {"id": 2, "title": "Read book", "done": True},        │
│    }                                                            │
│                                                                 │
│  Write:                                                         │
│  1. GET /tasks/{task_id}  → 404 if not found                    │
│  2. DELETE /tasks/{task_id}:                                    │
│       → 404 if task_id not in fake_tasks                        │
│       → 409 if the task is already done (done=True)             │
│       → 204 if deleted successfully                             │
│                                                                 │
│  3. What keyword — raise or return?                             │
│  4. What does the client receive in the body for a 204?         │
│  5. In DELETE, which check must come FIRST — 404 or 409?        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solution:**

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

fake_tasks: dict[int, dict] = {
    1: {"id": 1, "title": "Buy milk", "done": False},
    2: {"id": 2, "title": "Read book", "done": True},
}

@app.get("/tasks/{task_id}")
async def get_task(task_id: int):
    if task_id not in fake_tasks:
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")
    return fake_tasks[task_id]

@app.delete("/tasks/{task_id}", status_code=204)
async def delete_task(task_id: int):
    if task_id not in fake_tasks:           # 404 FIRST
        raise HTTPException(status_code=404, detail=f"Task {task_id} not found")
    if fake_tasks[task_id]["done"]:         # 409 SECOND
        raise HTTPException(status_code=409, detail="Cannot delete a completed task")
    del fake_tasks[task_id]
    return None   # 204 — FastAPI ignores this; body is always empty
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOLUTION EXPLAINED                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  raise — always. return HTTPException sends the object as JSON  │
│  with a 200 status. That is a bug, not an error response.       │
│                                                                 │
│  404 before 409: if task_id is missing, fake_tasks[task_id]     │
│  would raise a Python KeyError before you even reach the        │
│  done=True check. Check existence first, always.               │
│                                                                 │
│  204 body: empty. The client receives no content —              │
│  not even {} or null. return None is dead code here             │
│  but harmless.                                                  │
│                                                                 │
│  Tech debt note: @app.exception_handler for global error        │
│  formatting and RequestValidationError customisation            │
│  → Week 3, Lecture 4 (Error Handling & Dependencies)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: AUTOMATIC DOCUMENTATION

## 5.1 Swagger UI: Your API Playground (/docs)

**Every FastAPI app gets a free interactive API explorer.**

> "You have written zero documentation. Not a single line. Now open your browser."

```
http://127.0.0.1:8000/docs
```

**What you see:**

```
┌─────────────────────────────────────────────────────────────────┐
│              SWAGGER UI (/docs)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  📖 Bookstore API                                       │    │
│  │                                                         │    │
│  │  GET    /books              List Books          [▼]     │    │
│  │  POST   /books              Create Book         [▼]     │    │
│  │  GET    /books/{book_id}    Get Book             [▼]    │    │
│  │  PUT    /books/{book_id}    Update Book          [▼]    │    │
│  │  DELETE /books/{book_id}    Delete Book          [▼]    │    │
│  │                                                         │    │
│  │  Click any endpoint to expand:                          │    │
│  │  ├─ See all parameters with types                       │    │
│  │  ├─ See request body schema                             │    │
│  │  ├─ "Try it out" button → test the API live!            │    │
│  │  └─ See example responses                               │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  This is fully interactive. You can:                            │
│  ├─ Fill in parameters                                          │
│  ├─ Edit request bodies                                         │
│  ├─ Click "Execute"                                             │
│  ├─ See the actual HTTP request (curl command)                  │
│  └─ See the real response from your server                      │
│                                                                 │
│  No Postman. No curl. Test everything from the browser.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 ReDoc: The Readable Docs (/redoc)

**Same API, different presentation:**

```
http://127.0.0.1:8000/redoc
```

```
┌─────────────────────────────────────────────────────────────────┐
│                REDOC (/redoc)                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────┬─────────────────────────────────────┐    │
│  │  NAVIGATION       │  GET /books/{book_id}               │    │
│  │                   │                                     │    │
│  │  Books            │  Get a specific book by its ID.     │    │
│  │   GET /books      │                                     │    │
│  │   POST /books     │  PATH PARAMETERS                    │    │
│  │   GET /books/{id} │  ┌──────────┬──────┬──────────┐     │    │
│  │   PUT /books/{id} │  │ Name     │ Type │ Required │     │    │
│  │   DELETE /...     │  ├──────────┼──────┼──────────┤     │    │
│  │                   │  │ book_id  │ int  │ Yes      │     │    │
│  │  Schemas          │  └──────────┴──────┴──────────┘     │    │
│  │   BookCreate      │                                     │    │
│  │   BookUpdate      │  RESPONSES                          │    │
│  │                   │  200 — Successful Response          │    │
│  │                   │  422 — Validation Error             │    │
│  └───────────────────┴─────────────────────────────────────┘    │
│                                                                 │
│  Swagger UI → interactive playground (for developers)           │
│  ReDoc → clean documentation (for API consumers)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 How Type Hints Power Everything

**This is the central insight of the entire lecture.**

> "Where did all this documentation come from? You didn't write docs. You didn't configure Swagger. You didn't create JSON schemas. You wrote type hints, Pydantic models, and function signatures. FastAPI generated EVERYTHING else."

```python
@app.get("/authors/{author_id}/books")
async def get_author_books(
    author_id: int,                       # Docs: path param, type: integer
    sort_by: str = "title",               # Docs: query param, default: "title"
    limit: int = 20,                      # Docs: query param, default: 20
    include_drafts: bool = False,         # Docs: query param, default: false
) -> list[dict]:                          # Docs: response type
    """Get all books by a specific author.     ← Docs: endpoint description

    Retrieves the book catalog for the given   ← Docs: extended description
    author, with optional sorting and
    filtering.
    """
    ...
```

**Everything maps to documentation:**

```
┌─────────────────────────────────────────────────────────────────┐
│        YOUR CODE  →  GENERATED DOCUMENTATION                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Code                          │  Documentation                 │
│  ────────────────────          │  ─────────────────             │
│  @app.get("/path")             │  HTTP method + path            │
│  async def func_name()         │  Operation ID                  │
│  """docstring"""               │  Endpoint description          │
│  param: int                    │  Parameter type: integer       │
│  param: str = "default"        │  Default value shown           │
│  param: Optional[str] = None   │  Marked as optional            │
│  body: PydanticModel           │  Full JSON schema with fields  │
│  -> ReturnType                 │  Response schema               │
│                                                                 │
│                                                                 │
│  WEEK 1, LECTURE 1:   Type hints for mypy checking              │
│  WEEK 2, LECTURE 2:   Type hints for validation + docs          │
│                                                                 │
│  One syntax. Three purposes:                                    │
│  1. Static analysis (mypy catches bugs)                         │
│  2. Runtime validation (FastAPI rejects bad data)               │
│  3. Documentation (Swagger/ReDoc generated automatically)       │
│                                                                 │
│  Type hints aren't just annotation. They're infrastructure.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 The OpenAPI Schema (The Machine-Readable Contract)

**Behind Swagger and ReDoc is a JSON schema:**

```
http://127.0.0.1:8000/openapi.json
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   OPENAPI SCHEMA                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FastAPI generates an OpenAPI 3.1 specification automatically.  │
│                                                                 │
│  This is a standardized JSON document that describes:           │
│  ├─ Every endpoint in your API                                  │
│  ├─ Every parameter (path, query, body)                         │
│  ├─ Every data model (Pydantic → JSON Schema)                   │
│  ├─ Every response type                                         │
│  └─ Every possible error                                        │
│                                                                 │
│  Swagger UI reads this JSON and renders the interactive docs.   │
│  ReDoc reads the same JSON and renders the clean docs.          │
│                                                                 │
│                                                                 │
│  YOUR CODE                                                      │
│     │                                                           │
│     ▼                                                           │
│  FastAPI generates → openapi.json                               │
│                         │                                       │
│                    ┌────┴────┐                                  │
│                    ▼         ▼                                  │
│               Swagger UI   ReDoc                                │
│               (/docs)      (/redoc)                             │
│                                                                 │
│                                                                 │
│  Why this matters:                                              │
│  ├─ Frontend teams can read the schema and start coding         │
│  │   before your API is even finished                           │
│  ├─ API clients can be auto-generated from this schema          │
│  └─ Testing tools can use it for automated API testing          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
## 5.5 Path Operation Configuration

> "The `response_model` and `status_code` parameters go on the decorator. There is a whole family of additional decorator parameters that control how your endpoint appears in the auto-generated `/docs`. They are optional — but they matter the moment another developer opens your API."

```python
from fastapi import FastAPI

app = FastAPI()

@app.get(
    "/books",
    tags=["Books"],
    summary="List all books",
    description=(
        "Returns a paginated list of all books in the catalogue. "
        "Results are sorted by title by default."
    ),
    response_description="A list of book objects",
    status_code=200,
)
async def list_books(limit: int = 10):
    return []
```

**What each parameter does in `/docs`:**

```
┌─────────────────────────────────────────────────────────────────┐
│         PATH OPERATION PARAMETERS → EFFECT IN /docs             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  tags=["Books"]                                                 │
│  └─ Groups endpoints under a "Books" accordion in /docs.        │
│     All routes with the same tag appear together.               │
│     Convention: plural nouns — "Books", "Authors", "Orders"     │
│                                                                 │
│  summary="Short one-liner"                                      │
│  └─ Replaces the auto-generated function name in /docs.         │
│     Default: FastAPI converts list_books → "List Books"         │
│     With summary: shows your exact string.                      │
│                                                                 │
│  description="Long explanation..."                              │
│  └─ Shown when the user expands the endpoint card in /docs.     │
│     Supports markdown. Alternatively: use a docstring.          │
│                                                                 │
│  response_description="What 200 means"                          │
│  └─ Labels the 200 response section in /docs.                   │
│     Default text: "Successful Response"                         │
│                                                                 │
│  deprecated=True                                                │
│  └─ Shows a DEPRECATED badge in /docs. Endpoint still works.    │
│     Documentation only — no behaviour change.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Using a docstring instead of `description=`:**

```python
@app.get("/books", tags=["Books"], summary="List all books")
async def list_books(limit: int = 10):
    """
    List all books in the catalogue.

    Returns a paginated list sorted by title. Use the **limit**
    parameter to control how many results are returned at once.
    """
    return []
```

```
┌─────────────────────────────────────────────────────────────────┐
│              DOCSTRING vs description= PARAMETER                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Both set the long description in /docs. Pick one approach:     │
│                                                                 │
│  Docstring: conventional Python, visible in IDE hover,          │
│             good for long text with markdown.                   │
│                                                                 │
│  description=: all configuration in one place,                  │
│                easier to scan when reading the decorator.        │
│                                                                 │
│  Most codebases use docstrings. Be consistent within a project. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Marking an endpoint as deprecated:**

```python
@app.get(
    "/books/search",
    tags=["Books"],
    deprecated=True,
    summary="Search books (deprecated — use GET /books?query= instead)",
)
async def search_books(q: str):
    """This endpoint will be removed in v2. Use GET /books?query= instead."""
    return []
```

**Setting app-level metadata (applies to all of `/docs`):**

```python
app = FastAPI(
    title="Bookstore API",
    description="Manage your bookstore catalogue.",
    version="1.0.0",
)
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   FastAPI() PARAMETERS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  title=        → API name at the top of /docs                   │
│  description=  → Long description shown in /docs                │
│  version=      → Version string displayed in /docs              │
│                                                                 │
│  Every real API should set all three. They also appear in       │
│  openapi.json and are read by client code generators.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Adding `tags=` to every route takes 30 seconds per endpoint and saves hours of confusion the first time a new teammate opens your `/docs`. Make it a non-negotiable rule from the start."

---

# PART 6: FORMS AND FILES

## 6.1 When JSON Isn't Enough

> "Every request body you've written so far was JSON: `Content-Type: application/json`. But not every client speaks JSON. HTML forms submit data as `application/x-www-form-urlencoded` or `multipart/form-data`. File uploads require `multipart/form-data` — there is no alternative. If your API must accept a browser form submission or a file, JSON bodies won't work."

**You already know the format from the HTTP side:**

```
┌─────────────────────────────────────────────────────────────────┐
│            WHAT YOU ALREADY KNOW (Week 3, Lecture 1)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  In Lecture 1 you saw multipart/form-data from the client side: │
│  ├─ The boundary anatomy: how the body is split into parts      │
│  ├─ curl -F "field=value" and curl -F "file=@photo.jpg"         │
│  └─ httpx files= parameter for uploading files as a client      │
│                                                                 │
│  This lecture: the SERVER side. FastAPI reading that data.      │
│  This pays off the tech debt introduced in Lecture 1.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**First: install the required dependency:**

```bash
uv add python-multipart
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  ⚠ REQUIRED DEPENDENCY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  python-multipart is NOT bundled with FastAPI.                  │
│                                                                 │
│  Without it: Form() and UploadFile routes raise a runtime error:│
│  "Form data requires 'python-multipart' to be installed"        │
│                                                                 │
│  With it: form parsing and file uploads work.                   │
│                                                                 │
│  uv add python-multipart                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 Form Parameters (Form())

> "Form fields work like query parameters in code — but they come from the request body, not the URL."

```python
from fastapi import FastAPI, Form

app = FastAPI()

@app.post("/login")
async def login(
    username: str = Form(...),    # Required form field (... = no default)
    password: str = Form(...),    # Required form field
):
    return {"username": username, "authenticated": True}
```

**Test it:**

```bash
$ curl -X POST http://127.0.0.1:8000/login \
    -F "username=alice" \
    -F "password=secret123"

{"username":"alice","authenticated":true}
```

**How `Form(...)` works:**

```
┌─────────────────────────────────────────────────────────────────┐
│              FORM PARAMETERS — HOW THEY WORK                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Form(...) tells FastAPI:                                       │
│  "This parameter comes from the form body, not the query string"│
│                                                                 │
│  The ... (Ellipsis) means: required. No default.                │
│  Form("default") makes it optional with a fallback value.       │
│                                                                 │
│  FastAPI then:                                                  │
│  ├─ Checks Content-Type: multipart/form-data OR                 │
│  │  Content-Type: application/x-www-form-urlencoded             │
│  ├─ Extracts the named field from the form body                 │
│  ├─ Applies type coercion (same as path/query params)           │
│  └─ Returns 422 if a required field is missing                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The hard constraint — Form and JSON body are mutually exclusive:**

```
┌─────────────────────────────────────────────────────────────────┐
│             FORM vs JSON BODY — THEY CANNOT MIX                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ IMPOSSIBLE in one endpoint:                                 │
│  async def create(name: str = Form(...), item: ItemModel):      │
│  └─ Form() fields AND a Pydantic JSON body cannot coexist.      │
│     One request has one Content-Type. Pick one.                 │
│                                                                 │
│  ✅ FORM-only endpoint:                                         │
│  async def login(username: str = Form(...), pwd: str = Form(...))│
│                                                                 │
│  ✅ JSON-only endpoint:                                         │
│  async def create(item: ItemModel)                              │
│                                                                 │
│  ✅ FORM + FILE (same multipart body — covered in 6.5):         │
│  async def upload(name: str = Form(...), file: UploadFile)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Optional form fields:**

```python
from typing import Optional

@app.post("/profile")
async def update_profile(
    username: str           = Form(...),   # Required
    bio: Optional[str]      = Form(None),  # Optional, defaults to None
    age: int                = Form(18),    # Optional, defaults to 18
):
    return {"username": username, "bio": bio, "age": age}
```

> "Modern FastAPI documentation uses `Annotated[str, Form()]` syntax instead of `str = Form(...)`. Both are equivalent. This lecture uses `= Form(...)` for visual clarity. You will encounter `Annotated` throughout the official FastAPI docs and in contemporary codebases."

---

## 6.3 File Uploads (UploadFile)

> "File uploads use a different type: `UploadFile`. Unlike a form field that gives you a string, `UploadFile` is an async file-like object with metadata — the original filename the client sent, the MIME type, and an async read method."

```python
from fastapi import FastAPI, UploadFile

app = FastAPI()

@app.post("/upload", status_code=201)
async def upload_file(file: UploadFile):
    contents: bytes = await file.read()

    return {
        "filename":     file.filename,
        "content_type": file.content_type,
        "size_bytes":   len(contents),
    }
```

**Test it:**

```bash
$ curl -X POST http://127.0.0.1:8000/upload \
    -F "file=@/path/to/photo.jpg"

{"filename":"photo.jpg","content_type":"image/jpeg","size_bytes":84231}
```

**What `UploadFile` gives you:**

```
┌─────────────────────────────────────────────────────────────────┐
│              UploadFile ATTRIBUTES AND METHODS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  file.filename            → str | None                          │
│                              Original filename from the client. │
│                              None if the client didn't provide. │
│                                                                 │
│  file.content_type        → str | None                          │
│                              MIME type: "image/jpeg", "text/csv"│
│                              None if the client didn't provide. │
│                                                                 │
│  await file.read()        → bytes                               │
│                              Read all content. Entire file      │
│                              is loaded into memory at once.     │
│                                                                 │
│  await file.read(size)    → bytes                               │
│                              Read up to size bytes (chunked).   │
│                                                                 │
│  await file.seek(0)       → None                                │
│                              Rewind to the beginning. Required  │
│                              if you read() then want to read    │
│                              the file a second time.            │
│                                                                 │
│  await file.close()       → None                                │
│                              FastAPI calls this automatically   │
│                              after the request completes.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**`UploadFile` vs `bytes = File(...)` — two options:**

```python
from fastapi import File, UploadFile

# Option 1: UploadFile — async file object (recommended for most cases)
@app.post("/upload-async")
async def upload_async(file: UploadFile):
    contents = await file.read()
    return {"filename": file.filename, "size": len(contents)}


# Option 2: bytes = File(...) — reads entire file immediately as raw bytes
@app.post("/upload-bytes")
async def upload_bytes(file: bytes = File(...)):
    # Simpler — but no filename, no content_type, all in memory at once
    return {"size_bytes": len(file)}
```

```
┌─────────────────────────────────────────────────────────────────┐
│              UploadFile  vs  bytes = File(...)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│              UploadFile          bytes = File(...)              │
│              ────────────        ─────────────────              │
│  filename    ✅ Yes              ❌ Not available                │
│  content_type ✅ Yes             ❌ Not available                │
│  Large files ✅ Stream or chunk  ❌ All in memory immediately    │
│  Simplicity  ⚠️ await required   ✅ Just use len(file)          │
│                                                                 │
│  Default to UploadFile. Use bytes = File(...) only for small    │
│  files where filename and content_type don't matter.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   PRACTICE CHECKPOINT                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Write a POST /analyse endpoint that:                           │
│  - Accepts any file upload                                      │
│  - Returns: filename, content_type, size in bytes,              │
│    and the first 100 bytes as a list of integers                │
│    (hint: list(bytes_value[:100]) converts each byte to int)    │
│                                                                 │
│  Bonus: filename is Optional — the client might not send it.    │
│  Handle None gracefully in the response.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solution:**

```python
from fastapi import FastAPI, UploadFile

app = FastAPI()

@app.post("/analyse")
async def analyse_file(file: UploadFile):
    contents: bytes = await file.read()

    return {
        "filename":         file.filename     or "unknown",
        "content_type":     file.content_type or "application/octet-stream",
        "size_bytes":       len(contents),
        "first_100_bytes":  list(contents[:100]),
    }
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOLUTION EXPLAINED                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  file.filename is Optional[str]. Always guard against None —    │
│  some HTTP clients omit the filename entirely. Using `or`        │
│  provides a safe fallback without an if/else block.             │
│                                                                 │
│  list(contents[:100]): bytes support slicing like lists.        │
│  list() on a bytes value gives a list of integers (0–255),      │
│  one per byte. e.g. b'\x50\x44\x46' → [80, 68, 70]             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.4 Multiple File Uploads

```python
@app.post("/upload-many", status_code=201)
async def upload_many(files: list[UploadFile]):
    results = []
    for file in files:
        contents = await file.read()
        results.append({
            "filename":   file.filename,
            "size_bytes": len(contents),
        })
    return {"uploaded": len(files), "files": results}
```

**Test it:**

```bash
$ curl -X POST http://127.0.0.1:8000/upload-many \
    -F "files=@photo1.jpg" \
    -F "files=@photo2.png" \
    -F "files=@document.pdf"

{"uploaded":3,"files":[...]}
```

> "Same field name repeated in the curl command. FastAPI collects all parts with the same name and assembles them into the `list[UploadFile]`."

---

## 6.5 Form + File Combined

> "Form fields and file uploads can coexist in the same endpoint because both are parts of a `multipart/form-data` body. What you cannot do is mix either of them with a Pydantic JSON body."

```python
from fastapi import FastAPI, Form, File, UploadFile
from typing import Optional

app = FastAPI()

@app.post("/documents", status_code=201)
async def upload_document(
    title:       str            = Form(...),
    category:    str            = Form(...),
    description: Optional[str]  = Form(None),
    file:        UploadFile     = File(...),   # File() makes it explicit + required
):
    contents = await file.read()
    return {
        "title":        title,
        "category":     category,
        "description":  description,
        "filename":     file.filename,
        "content_type": file.content_type,
        "size_bytes":   len(contents),
    }
```

**Test it:**

```bash
$ curl -X POST http://127.0.0.1:8000/documents \
    -F "title=Q3 Report" \
    -F "category=finance" \
    -F "description=Quarterly summary" \
    -F "file=@report.pdf"
```

**What's happening in the raw request:**

```
┌─────────────────────────────────────────────────────────────────┐
│           WHAT'S IN THE MULTIPART BODY (from W3L1)              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Content-Type: multipart/form-data; boundary=----XYZ            │
│                                                                 │
│  ----XYZ                                                        │
│  Content-Disposition: form-data; name="title"                   │
│  Q3 Report                                                      │
│  ----XYZ                                                        │
│  Content-Disposition: form-data; name="category"                │
│  finance                                                        │
│  ----XYZ                                                        │
│  Content-Disposition: form-data; name="file"; filename="r.pdf"  │
│  Content-Type: application/pdf                                  │
│  [binary PDF bytes]                                             │
│  ----XYZ--                                                      │
│                                                                 │
│  FastAPI reads: title from the "title" part                     │
│                 category from the "category" part               │
│                 file from the binary part → UploadFile          │
│                                                                 │
│  This is the boundary anatomy from W3 Lecture 1. Now you're     │
│  reading it on the server, not just sending it from a client.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                       TECH DEBT NOTE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Deferred from this lecture:                                    │
│  ├─ Validating file size before reading the entire file         │
│  ├─ Chunked / streaming reads for very large files              │
│  ├─ Saving files to disk with aiofiles (async file writes)      │
│  └─ Cloud storage: S3, GCS, Cloudflare R2 upload patterns       │
│                                                                 │
│  → Paid off: Week 13 (Capstone)                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 7: HEADERS AND COOKIES

## 7.1 Reading Request Headers (Header())

> "In Week 3 Lecture 1 you saw headers in raw HTTP — `User-Agent`, `Authorization`, `Content-Type`. Now you'll read them in FastAPI routes. The syntax is nearly identical to query parameters: declare a function parameter with the right type and `Header()`."

```python
from fastapi import FastAPI, Header
from typing import Optional

app = FastAPI()

@app.get("/client-info")
async def client_info(
    user_agent: Optional[str] = Header(default=None),
    accept:     Optional[str] = Header(default=None),
):
    return {
        "user_agent": user_agent,
        "accept":     accept,
    }
```

**Test it:**

```bash
$ curl http://127.0.0.1:8000/client-info \
    -H "Accept: application/json"

{"user_agent":"curl/8.1.2","accept":"application/json"}
```

```
┌─────────────────────────────────────────────────────────────────┐
│           HEADER NAMING — THE UNDERSCORE RULE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP headers use hyphens:     User-Agent   X-Api-Key           │
│  Python parameters use underscores: user_agent  x_api_key       │
│                                                                 │
│  FastAPI AUTOMATICALLY converts underscores → hyphens:          │
│                                                                 │
│  Python parameter      HTTP header FastAPI reads                │
│  ─────────────────      ─────────────────────────               │
│  user_agent         →  User-Agent                               │
│  x_api_key          →  X-Api-Key                                │
│  x_request_id       →  X-Request-Id                             │
│  content_type       →  Content-Type                             │
│                                                                 │
│  You can NEVER write user-agent in Python (invalid identifier). │
│  Write user_agent — FastAPI reads User-Agent for you.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Custom API key header — a preview of auth patterns:**

```python
from fastapi import HTTPException

VALID_API_KEY = "my-secret-key"

@app.get("/protected")
async def protected_route(
    x_api_key: Optional[str] = Header(default=None),
):
    if x_api_key != VALID_API_KEY:
        raise HTTPException(status_code=401, detail="Invalid or missing API key")
    return {"message": "Access granted"}
```

> "This is a preview. The full authentication system with JWT, token refresh, and RBAC comes in Week 9. But even this simple `X-Api-Key` pattern shows how credentials travel in headers — which is exactly where they belong."

```
┌─────────────────────────────────────────────────────────────────┐
│                   PRACTICE CHECKPOINT                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Write a GET /greet endpoint that:                              │
│  - Reads an X-User-Name header                                  │
│  - Defaults to "stranger" if the header is absent               │
│  - Returns {"message": "Hello, {name}!"}                        │
│                                                                 │
│  Then answer:                                                   │
│  1. What is the Python parameter name?                          │
│  2. What exact HTTP header does FastAPI look for in the request? │
│  3. Write a curl command that sends the header and triggers     │
│     a personalised greeting.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solution:**

```python
from fastapi import FastAPI, Header

app = FastAPI()

@app.get("/greet")
async def greet(
    x_user_name: str = Header(default="stranger"),
    # Python:    x_user_name  (underscore)
    # HTTP:      X-User-Name  (hyphen, case-insensitive)
):
    return {"message": f"Hello, {x_user_name}!"}
```

```bash
$ curl http://127.0.0.1:8000/greet -H "X-User-Name: Alice"
{"message":"Hello, Alice!"}

$ curl http://127.0.0.1:8000/greet
{"message":"Hello, stranger!"}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOLUTION EXPLAINED                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Python parameter: x_user_name                                  │
│  HTTP header read: X-User-Name                                  │
│                                                                 │
│  The conversion is automatic. You never write x-user-name       │
│  in Python code because it is not a valid identifier.           │
│                                                                 │
│  HTTP headers are case-insensitive (RFC 7230). FastAPI          │
│  normalises to lowercase internally. X-User-Name, x-user-name, │
│  X-USER-NAME all match the same parameter.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.2 Setting Response Headers

> "Reading request headers uses `Header()`. Setting response headers uses a different approach: FastAPI injects a mutable `Response` object when it sees `Response` as a type annotation — the same type-based injection pattern as `Request`."

```python
from fastapi import FastAPI, Response

app = FastAPI()

@app.get("/items/{item_id}")
async def get_item(item_id: int, response: Response):
    # Mutate the response headers — FastAPI picks up the changes
    response.headers["X-Item-Id"]   = str(item_id)
    response.headers["Cache-Control"] = "max-age=300"
    response.headers["X-Served-By"]  = "fastapi"

    # Return a dict as normal — serialisation is still automatic
    return {"item_id": item_id, "title": "Some item"}
```

**Test it:**

```bash
$ curl -I http://127.0.0.1:8000/items/42

HTTP/1.1 200 OK
x-item-id: 42
cache-control: max-age=300
x-served-by: fastapi
content-type: application/json
```

```
┌─────────────────────────────────────────────────────────────────┐
│           RESPONSE INJECTION — KEY FACTS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  response: Response does NOT change your return value.          │
│  You still return a dict; FastAPI still converts it to JSON.    │
│  The Response injection only affects headers (and status).      │
│                                                                 │
│  response.headers["X-Key"] = "value"  → sets one header         │
│  response.status_code = 201           → overrides status code   │
│                                          (rarely needed here)   │
│                                                                 │
│  This differs from returning JSONResponse(...) directly.        │
│  Injecting Response gives you header control WITHOUT giving     │
│  up FastAPI's automatic serialisation and response_model.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.3 Reading Cookies (Cookie())

> "Cookies are name-value pairs that browsers store and send automatically with every request to the same origin. FastAPI reads them with `Cookie()` — same pattern as `Header()` and `Form()`."

```python
from fastapi import FastAPI, Cookie
from typing import Optional

app = FastAPI()

@app.get("/me")
async def get_current_user(
    session_token: Optional[str] = Cookie(default=None),
):
    if session_token is None:
        return {"user": "anonymous"}
    # In a real app: decode token, look up user in DB
    return {"session_token": session_token, "user": "authenticated"}
```

**Test it:**

```bash
$ curl http://127.0.0.1:8000/me \
    --cookie "session_token=abc123"

{"session_token":"abc123","user":"authenticated"}
```

```
┌─────────────────────────────────────────────────────────────────┐
│              COOKIES — WHAT IS DEFERRED                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Cookie() reads a cookie value FROM the request.                │
│  That is all you need to know for now.                          │
│                                                                 │
│  Setting cookies IN the response:                               │
│  response.set_cookie(key="session_token", value="abc123")       │
│                                                                 │
│  Setting cookies with security flags:                           │
│  response.set_cookie(                                           │
│      key="session_token",                                       │
│      value="abc123",                                            │
│      httponly=True,    ← browser JS cannot read this cookie     │
│      secure=True,      ← HTTPS only                             │
│      samesite="lax",   ← CSRF protection                        │
│  )                                                              │
│                                                                 │
│  In Lecture 1 you saw these flags from the HTTP protocol side   │
│  (section 5.3). The FastAPI implementation of secure            │
│  cookie-based auth → Week 9, Lectures 1–3.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 8: PROJECT STRUCTURE

## 8.1 When One File Isn't Enough

> "Right now everything fits in `main.py`. But the Task Manager project has tasks, categories, and tags — each needing CRUD endpoints. Five resources × five operations = 25 route functions. Add Pydantic models for each, and `main.py` hits 400 lines before Week 4 ends. By Week 9, with auth, it would be 1500 lines. You'd spend more time navigating the file than writing code."

```
┌─────────────────────────────────────────────────────────────────┐
│              SIGNS YOU NEED TO SPLIT YOUR ROUTES                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ├─ main.py is approaching 150+ lines                           │
│  ├─ Multiple unrelated resources in one file                    │
│  │   (tasks AND categories AND tags in one file)                │
│  ├─ Two teammates constantly editing the same file —            │
│  │   merge conflicts every single day                           │
│  └─ "I can't find the endpoint I'm looking for" in your own     │
│     codebase after two weeks of writing it                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.2 APIRouter for Modular Routes

> "`APIRouter` is a FastAPI object that works exactly like `app` — but lives in its own file. It collects route registrations and carries them back to `main.py` when you include it."

```python
# routers/books.py

from fastapi import APIRouter, HTTPException

router = APIRouter(
    prefix="/books",   # Every route in this file is prefixed with /books
    tags=["Books"],    # Every route grouped under "Books" in /docs
)

# Use @router — NOT @app — inside a router file
@router.get("/")
async def list_books():
    return [{"id": 1, "title": "Dune"}]

@router.get("/{book_id}")
async def get_book(book_id: int):
    return {"book_id": book_id}

@router.post("/", status_code=201)
async def create_book():
    return {"id": 2, "title": "New Book"}

@router.delete("/{book_id}", status_code=204)
async def delete_book(book_id: int):
    return None
```

```
┌─────────────────────────────────────────────────────────────────┐
│           APIRouter IMPORTANT DIFFERENCE FROM app               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @app.get("/books")    → full path declared in main.py          │
│  @router.get("/")      → just "/" — prefix comes from the       │
│                          APIRouter() declaration                │
│                                                                 │
│  With prefix="/books":                                          │
│  @router.get("/")          → GET  /books/                       │
│  @router.get("/{id}")      → GET  /books/{id}                   │
│  @router.post("/")         → POST /books/                       │
│  @router.delete("/{id}")   → DELETE /books/{id}                 │
│                                                                 │
│  The prefix is applied ONCE in APIRouter() — not on each route. │
│  Every route in the file inherits it automatically.             │
│                                                                 │
│  ❌ COMMON MISTAKE:                                             │
│  @app.get("/") inside routers/books.py                          │
│  ↑ Using @app instead of @router. Route is invisible to the     │
│  include_router system and will not appear in the running app.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.3 Including Routers (include_router)

**Register each router with the app in `main.py`:**

```python
# main.py

from fastapi import FastAPI
from routers import books, authors, orders

app = FastAPI(
    title="Bookstore API",
    version="1.0.0",
)

app.include_router(books.router)
app.include_router(authors.router)
app.include_router(orders.router)
```

**Prefix stacking — how prefixes combine at include time:**

```python
# Option A: prefix lives in the router file (default approach)
router = APIRouter(prefix="/books")
app.include_router(books.router)           # → /books/...

# Option B: no prefix in router, added at include time
router = APIRouter()
app.include_router(books.router, prefix="/books")  # → /books/...

# Option C: prefix in BOTH — they STACK
router = APIRouter(prefix="/books")
app.include_router(books.router, prefix="/api/v1") # → /api/v1/books/...
```

```
┌─────────────────────────────────────────────────────────────────┐
│              PREFIX STACKING — FINAL URL CALCULATION            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Final URL = include_router prefix + APIRouter prefix + route   │
│                                                                 │
│  include_router(books.router, prefix="/api/v1")                 │
│  + APIRouter(prefix="/books")                                   │
│  + @router.get("/{book_id}")                                    │
│  ─────────────────────────────────────                          │
│  = GET /api/v1/books/{book_id}                                  │
│                                                                 │
│  This is how API versioning works in practice.                  │
│  Your router files stay unchanged. You version at include time. │
│  Full versioning implementation → Week 4, Lecture 2             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.4 Typical Project Layout

```
task_manager/
├── main.py                  ← App creation + include_router only
├── routers/                 ← One file per resource group
│   ├── __init__.py          ← Empty file — makes it a Python package
│   ├── tasks.py             ← /tasks/* endpoints
│   ├── categories.py        ← /categories/* endpoints
│   └── tags.py              ← /tags/* endpoints
├── schemas/                 ← Pydantic input/output models
│   ├── __init__.py
│   ├── task.py              ← TaskCreate, TaskResponse, TaskUpdate
│   └── category.py
└── tests/
    ├── conftest.py
    ├── test_tasks.py
    └── test_categories.py
```

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY schemas/ IS SEPARATE FROM routers/             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pydantic models are DATA SHAPES — they don't belong in the     │
│  routing layer. They belong in their own schemas/ layer.        │
│                                                                 │
│  # routers/tasks.py imports from schemas/                       │
│  from schemas.task import TaskCreate, TaskResponse              │
│                                                                 │
│  This separation means:                                         │
│  ├─ Multiple routers can share the same Pydantic model          │
│  ├─ You update a schema without touching routing logic          │
│  └─ As the codebase grows, models/ (SQLAlchemy — Week 5+)       │
│     and schemas/ (Pydantic) stay clearly distinct               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What `main.py` now looks like:**

```python
# main.py — thin by design
from fastapi import FastAPI
from routers import tasks, categories, tags

app = FastAPI(
    title="Task Manager API",
    description="Manage tasks, categories, and tags.",
    version="0.1.0",
)

app.include_router(tasks.router)
app.include_router(categories.router)
app.include_router(tags.router)
```

> "Notice how thin `main.py` has become: it creates the app, registers routers, and nothing else. All route logic lives in the router files. All data shapes live in schemas. This is the structure professional FastAPI projects use. You will apply it in the Task Manager project starting this week."

```
┌─────────────────────────────────────────────────────────────────┐
│                   PRACTICE CHECKPOINT                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You have a main.py with 16 routes: 4 for tasks, 4 for          │
│  categories, 4 for tags, 4 for users.                           │
│                                                                 │
│  1. Draw the directory structure you need to create.            │
│  2. Write routers/tasks.py with prefix and tags set.            │
│     Include GET /tasks (list) and POST /tasks (create)          │
│     with correct status codes. Stub returns are fine.           │
│  3. Write the full main.py (imports + include_router only).     │
│  4. The router has prefix="/tasks". You call:                   │
│     app.include_router(tasks.router, prefix="/api/v1")          │
│     What is the final URL for POST tasks?                       │
│  5. Where do TaskCreate and TaskResponse Pydantic models live?  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solution:**

```
task_manager/
├── main.py
├── routers/
│   ├── __init__.py       ← empty
│   ├── tasks.py
│   ├── categories.py
│   ├── tags.py
│   └── users.py
└── schemas/
    ├── __init__.py       ← empty
    └── task.py           ← TaskCreate, TaskResponse, TaskUpdate
```

```python
# routers/tasks.py
from fastapi import APIRouter

router = APIRouter(
    prefix="/tasks",
    tags=["Tasks"],
)

@router.get("/")
async def list_tasks():
    return []

@router.post("/", status_code=201)
async def create_task():
    return {"id": 1, "title": "New task"}
```

```python
# main.py
from fastapi import FastAPI
from routers import tasks, categories, tags, users

app = FastAPI(title="Task Manager API", version="0.1.0")

app.include_router(tasks.router)
app.include_router(categories.router)
app.include_router(tags.router)
app.include_router(users.router)
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOLUTION EXPLAINED                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q4: Final URL for POST tasks with stacked prefixes:            │
│                                                                 │
│  include_router prefix:  /api/v1                                │
│  APIRouter prefix:        /tasks                                │
│  Route:                   /                                     │
│  ─────────────────────────────────                              │
│  Final URL:               POST /api/v1/tasks/                   │
│                                                                 │
│  Q5: Pydantic models live in schemas/task.py, not in the        │
│  router. The router imports them:                               │
│  from schemas.task import TaskCreate, TaskResponse              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 9: BEYOND JSON RESPONSES

## 9.1 Response Classes Overview

> "Every response you have returned so far has been JSON — FastAPI serialises your dict automatically. But APIs occasionally need to return other things: an HTML page, a file download, a redirect, plain text. FastAPI provides typed response classes for each case."

```python
from fastapi import FastAPI
from fastapi.responses import (
    HTMLResponse,
    PlainTextResponse,
    FileResponse,
    RedirectResponse,
)

app = FastAPI()

# Return HTML
@app.get("/page", response_class=HTMLResponse)
async def html_page():
    return "<html><body><h1>Hello from FastAPI</h1></body></html>"

# Return plain text — common for health checks
@app.get("/health", response_class=PlainTextResponse)
async def health_check():
    return "OK"

# Return a file for download
@app.get("/report")
async def download_report():
    return FileResponse(
        path="report.pdf",
        filename="monthly_report.pdf",   # Sets Content-Disposition header
        media_type="application/pdf",
    )

# Redirect to another URL
@app.get("/old-path")
async def old_endpoint():
    return RedirectResponse(url="/new-path", status_code=301)
```

---

## 9.2 When to Use Each

```
┌─────────────────────────────────────────────────────────────────┐
│              RESPONSE CLASSES — WHEN TO USE WHAT                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JSON (default)                                                 │
│  └─ Your API endpoints. 95%+ of everything you will write.      │
│     return {"key": "value"} — FastAPI handles it automatically. │
│                                                                 │
│  HTMLResponse                                                   │
│  └─ When FastAPI serves HTML directly (rare in pure API         │
│     projects — usually a frontend framework handles HTML).      │
│     Health dashboards, lightweight admin pages.                 │
│                                                                 │
│  PlainTextResponse                                              │
│  └─ /health endpoints. Webhook verification. robots.txt.        │
│     Anywhere that HTML is overkill.                             │
│                                                                 │
│  FileResponse                                                   │
│  └─ File downloads: PDFs, CSVs, data exports.                   │
│     Sets Content-Disposition so the browser saves the file.     │
│                                                                 │
│  RedirectResponse                                               │
│  └─ Old URL → new URL (301 Permanent).                          │
│     POST-redirect-GET pattern (302/303).                        │
│     HTTP → HTTPS redirect.                                      │
│                                                                 │
│  StreamingResponse                                              │
│  └─ Very large files, server-sent events, real-time data.       │
│     → Deferred: Week 12 (already in tech debt from W3L1)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Controlling the default response class for the whole app:**

```python
from fastapi.responses import ORJSONResponse

# Replace JSONResponse with ORJSONResponse globally
# (faster JSON serialisation via the orjson library)
app = FastAPI(default_response_class=ORJSONResponse)
```

> "The default JSON response class is fine for now. Once you run load tests in Week 12, you will see whether `ORJSONResponse` actually helps your specific workload. Measure before you optimise."

---

# UPDATED QUICK REFERENCE CARD — ADDITIONS

> *Append these entries to the existing Quick Reference Card.*

```
┌─────────────────────────────────────────────────────────────────┐
│          QUICK REFERENCE — ADDITIONS FROM THIS LECTURE          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STATUS CODES:                                                  │
│      @app.post("/items",       status_code=201)                 │
│      @app.delete("/items/{id}",status_code=204)                 │
│      from fastapi import status                                 │
│      status.HTTP_201_CREATED   status.HTTP_204_NO_CONTENT       │
│                                                                 │
│  RESPONSE MODEL:                                                │
│      @app.post("/items", response_model=ItemResponse,           │
│                          status_code=201)                       │
│      @app.patch("/items/{id}", response_model=ItemUpdate,       │
│                response_model_exclude_unset=True)               │
│                                                                 │
│  HTTP EXCEPTION:                                                │
│      from fastapi import HTTPException                          │
│      raise HTTPException(status_code=404, detail="Not found")   │
│      raise HTTPException(status_code=409,                       │
│                          detail={"error": "conflict"})          │
│      # Use raise — NOT return                                   │
│                                                                 │
│  RAW REQUEST OBJECT:                                            │
│      from fastapi import Request                                │
│      async def route(request: Request, q: int = 0):             │
│          ip = request.client.host                               │
│          ua = request.headers.get("user-agent")                 │
│      # Request must come before defaulted params in signature   │
│                                                                 │
│  FORM PARAMETERS:                                               │
│      from fastapi import Form                                   │
│      async def f(field: str = Form(...)):  # required           │
│      async def f(field: str = Form("default")):  # optional     │
│      # Requires: uv add python-multipart                        │
│      # Cannot mix Form() and Pydantic JSON body in one endpoint │
│                                                                 │
│  FILE UPLOADS:                                                  │
│      from fastapi import File, UploadFile                       │
│      async def upload(file: UploadFile):                        │
│          contents = await file.read()                           │
│          file.filename       # str | None                       │
│          file.content_type   # str | None                       │
│      async def multi(files: list[UploadFile]): ...              │
│      async def combined(title: str = Form(...),                 │
│                         file: UploadFile = File(...)): ...      │
│                                                                 │
│  REQUEST HEADERS:                                               │
│      from fastapi import Header                                 │
│      from typing import Optional                                │
│      async def route(user_agent: Optional[str]=Header(None)):   │
│      # underscore in Python → hyphen in HTTP header name        │
│                                                                 │
│  RESPONSE HEADERS:                                              │
│      from fastapi import Response                               │
│      async def route(response: Response):                       │
│          response.headers["X-Custom"] = "value"                 │
│      # Return dict as normal — serialisation unchanged          │
│                                                                 │
│  COOKIES:                                                       │
│      from fastapi import Cookie                                 │
│      async def route(session: Optional[str]=Cookie(None)):      │
│      # Setting cookies (setting flags deferred → Week 9)        │
│      # response.set_cookie(key="k", value="v")                  │
│                                                                 │
│  APIROUTER:                                                     │
│      from fastapi import APIRouter                              │
│      router = APIRouter(prefix="/items", tags=["Items"])         │
│      @router.get("/")   @router.post("/", status_code=201)      │
│      # In main.py:                                              │
│      app.include_router(items.router)                           │
│      app.include_router(items.router, prefix="/api/v1")         │
│      # Final URL = include prefix + router prefix + route       │
│                                                                 │
│  PATH OPERATION CONFIG:                                         │
│      @app.get("/items",                                         │
│               tags=["Items"],                                   │
│               summary="List items",                             │
│               description="Long description here...",           │
│               response_description="List of item objects",      │
│               deprecated=True)                                  │
│      app = FastAPI(title="My API", version="1.0.0")             │
│                                                                 │
│  NON-JSON RESPONSES:                                            │
│      from fastapi.responses import (                            │
│          HTMLResponse, PlainTextResponse,                       │
│          FileResponse, RedirectResponse)                        │
│      return FileResponse(path="f.pdf", filename="dl.pdf",       │
│                          media_type="application/pdf")          │
│      return RedirectResponse(url="/new", status_code=301)       │
│      @app.get("/health", response_class=PlainTextResponse)      │
│      async def health(): return "OK"                            │
│                                                                 │
│  ADDITIONAL COMMON MISTAKES:                                    │
│      ❌ return HTTPException(...)  → must RAISE, not return      │
│      ❌ Form() + Pydantic body     → mutually exclusive          │
│      ❌ @app.get inside router file → use @router.get            │
│      ❌ Forgetting python-multipart → Form/File runtime error    │
│      ❌ status_code=204 with body  → body is silently dropped    │
│      ❌ Defaulted param before Request → Python SyntaxError      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# UPDATED: CONNECTION TO UPCOMING LECTURES

> *Replace the existing "Connection to Upcoming Lectures" box with the following.*

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 3, LECTURE 3 (Next — Pydantic Deep Dive):                 │
│  └─ You saw BaseModel basics and response_model filtering.       │
│     Next: Field constraints (Field, gt, lt, min_length,         │
│     regex), nested models, custom validators                    │
│     (@field_validator, @model_validator), model_config,         │
│     response_model_include/exclude sets, serialisation control  │
│     (exclude, include, by_alias).                               │
│     BaseModel goes from "typed dict" to "full schema engine."   │
│                                                                 │
│  WEEK 3, LECTURE 4 (Error Handling & Dependencies):             │
│  └─ You saw basic HTTPException raise patterns.                 │
│     Next: @app.exception_handler for global error formatting,   │
│     RequestValidationError customisation, Depends() for         │
│     shared logic (DB sessions, auth checks), yield              │
│     dependencies for setup/teardown, dependency scope and       │
│     caching, APIRouter router-level dependencies.               │
│     Connect to context managers (Week 1, Lecture 2).            │
│                                                                 │
│  WEEK 4, LECTURE 1 (Testing FastAPI):                           │
│  └─ How to test every endpoint you built in Lectures 2–4.       │
│     TestClient for JSON, form fields, file uploads,             │
│     custom headers. Test that response_model filters output.    │
│     Test that HTTPException returns the right status code.      │
│     Test APIRouter-structured apps with conftest.py.            │
│                                                                 │
│  WEEK 4, LECTURE 2 (API Design Principles):                     │
│  └─ APIRouter prefix stacking becomes full API versioning:      │
│     deprecation headers, sunset dates, maintaining two active   │
│     versions simultaneously.                                    │
│                                                                 │
│  WEEK 9 (Authentication):                                       │
│  └─ Cookie() for reading becomes response.set_cookie() with     │
│     HttpOnly, Secure, SameSite flags. JWT in cookies vs         │
│     localStorage. The auth patterns build directly on Header()  │
│     and Cookie() from Part 7 of this lecture.                   │
│                                                                 │
│  WEEK 12 (Real-Time & Performance):                             │
│  └─ StreamingResponse and EventSourceResponse (tech debt from   │
│     W3L1). The response class pattern from Part 9 extends       │
│     naturally to streaming. GzipMiddleware is also paid off.    │
│                                                                 │
│  WEEK 13 (Capstone):                                            │
│  └─ UploadFile storage: saving to disk (aiofiles), uploading    │
│     to S3/cloud storage, file size validation before reading.   │
│                                                                 │
│  THE PATTERNS YOU LEARNED TODAY CARRY YOU:                      │
│  Path + Query + Body + Status Code + Response Model +           │
│  HTTPException + APIRouter. Every future lecture adds           │
│  business logic and integrations inside routes — the shape      │
│  of the routes themselves does not change from here.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

