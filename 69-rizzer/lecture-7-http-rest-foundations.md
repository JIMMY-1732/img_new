# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REVEAL THE INVISIBLE                                           │
│  ────────────────────                                           │
│  Students use HTTP hundreds of times a day without knowing it.  │
│  Pull back the curtain. Show them the raw text on the wire.     │
│                                                                 │
│  PROTOCOL BEFORE FRAMEWORK                                      │
│  ─────────────────────────                                      │
│  Next lecture introduces FastAPI. If students don't understand  │
│  HTTP first, they'll memorize decorators without understanding  │
│  what @app.get actually means. That's not learning.             │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  HTTP is abstract until mapped to something tangible.           │
│  We use a library front desk analogy throughout.                │
│  Every concept maps to a physical interaction.                  │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Dataclasses  → JSON objects are the same shape                 │
│  Exceptions   → Status codes are exception hierarchies          │
│  Async        → HTTP requests are the I/O that async handles    │
│  Type hints   → Structured metadata, structured contracts       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                   HTTP & REST FOUNDATIONS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE INVISIBLE CONVERSATION (30 min)                    │
│  ├─ 1.1 The Question You Can't Answer Yet                       │
│  ├─ 1.2 Pulling Back the Curtain                                │
│  ├─ 1.3 HTTP is Just Text                                       │
│  └─ 1.4 The Library Analogy                                     │
│                                                                 │
│  PART 2: ANATOMY OF HTTP                                        │
│  ├─ 2.1 The Request — What You Send                             │
│  ├─ 2.2 The Response — What Comes Back                          │
│  ├─ 2.3 The Full Cycle (Visual)                                 │
│  ├─ 2.4 Seeing It Live (curl, DevTools, and Python)             │
│  └─ 2.5 The HTTP Evolution (HTTP/1.1 → HTTP/2 → HTTP/3)        │
│                                                                 │
│  PART 3: HTTP METHODS — THE FIVE VERBS (45 min)                 │
│  ├─ 3.1 Why Multiple Methods?                                   │
│  ├─ 3.2 GET — Retrieve a Resource                               │
│  ├─ 3.3 POST — Create a Resource                                │
│  ├─ 3.4 PUT vs PATCH — Replace vs Update                        │
│  ├─ 3.5 DELETE — Remove a Resource                              │
│  └─ 3.6 Safety, Idempotency, and Idempotency Keys               │
│                                                                 │
│  PART 4: STATUS CODES — THE SERVER'S VOCABULARY                 │
│  ├─ 4.1 The Five Families                                       │
│  ├─ 4.2 2xx — It Worked                                         │
│  ├─ 4.3 3xx — Look Elsewhere                                    │
│  ├─ 4.4 4xx — You Messed Up                                     │
│  ├─ 4.5 5xx — We Messed Up                                      │
│  └─ 4.6 Status Codes as Exceptions + Error Response Structure   │
│                                                                 │
│  PART 5: HEADERS — THE METADATA LAYER                           │
│  ├─ 5.1 What Are Headers?                                       │
│  ├─ 5.2 Content-Type (The Language Agreement)                   │
│  ├─ 5.3 Authorization, Cookies, and Identity                    │
│  ├─ 5.4 Other Essential Headers                                 │
│  ├─ 5.5 Custom Headers                                          │
│  └─ 5.6 HTTPS and TLS — The Security Layer                      │
│                                                                 │
│  PART 6: REST — THE ARCHITECTURE                                │
│  ├─ 6.1 What REST Actually Means                                │
│  ├─ 6.2 Everything is a Resource                                │
│  ├─ 6.3 URLs as Resource Addresses                              │
│  ├─ 6.4 Statelessness                                           │
│  ├─ 6.5 Uniform Interface                                       │
│  ├─ 6.6 Good vs Bad API Design                                  │
│  └─ 6.7 CORS — Why Browsers Block Your Requests                 │
│                                                                 │
│  PART 7: JSON — THE DATA FORMAT                                 │
│  ├─ 7.1 Why JSON Won                                            │
│  ├─ 7.2 JSON Syntax Rules                                       │
│  ├─ 7.3 Python ↔ JSON (The json Module)                         │
│  ├─ 7.4 Dataclasses to JSON (Connection to Week 1)              │
│  ├─ 7.5 The Serialization Pipeline (Preview of Pydantic)        │
│  └─ 7.6 File Uploads — When JSON Isn't Enough                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE INVISIBLE CONVERSATION

## 1.1 The Question You Can't Answer Yet

**Start with a sneak peek. Show them where they're headed.**

```python
# You will write this NEXT LECTURE. Don't worry about the syntax.
# Focus on the QUESTIONS.

from fastapi import FastAPI

app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"id": user_id, "name": "Alice"}
```

**Now ask the class:**

> "You'll write code exactly like this next lecture. But before you do — can you answer these questions?"

```
┌─────────────────────────────────────────────────────────────────┐
│                  CAN YOU ANSWER THESE?                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. What does "GET" mean in @app.get()?                         │
│     Why not just @app.do() or @app.handle()?                    │
│                                                                 │
│  2. What does "/users/{user_id}" represent?                     │
│     Why this format? Who decided?                               │
│                                                                 │
│  3. What ACTUALLY travels over the network when someone         │
│     calls this endpoint? What does the request look like?       │
│                                                                 │
│  4. Why does it return a dict? What does the caller             │
│     actually receive?                                           │
│                                                                 │
│  5. What happens if user_id doesn't exist?                      │
│     What does the response look like?                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The honest answer:** You can't answer most of these yet. And that's the problem.

> "If you use FastAPI without understanding HTTP, you're copying syntax without understanding what's happening. You'll write code that works by accident and breaks in ways you can't debug. This lecture is about making sure that NEVER happens."

---

## 1.2 Pulling Back the Curtain

**What ACTUALLY happens when someone calls that endpoint?**

Let's trace the entire journey. A client (browser, mobile app, another server) wants to get information about user #42.

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT ACTUALLY HAPPENS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: Client opens a network connection to the server        │
│          (TCP connection — just a pipe for sending bytes)       │
│                                                                 │
│  Step 2: Client sends TEXT through the pipe:                    │
│                                                                 │
│          GET /users/42 HTTP/1.1                                 │
│          Host: api.example.com                                  │
│          Accept: application/json                               │
│                                                                 │
│  Step 3: Server reads the text, understands the request,        │
│          finds user #42, and sends TEXT back:                   │
│                                                                 │
│          HTTP/1.1 200 OK                                        │
│          Content-Type: application/json                         │
│                                                                 │
│          {"id": 42, "name": "Alice"}                            │
│                                                                 │
│  Step 4: Client reads the response text, parses it,             │
│          and does something with the data.                      │
│                                                                 │
│  Step 5: Connection closes.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Pause. Look at Step 2 and Step 3 again.**

> "That's it. That's HTTP. It's TEXT. Human-readable, plain text being sent back and forth over a network connection. Not binary. Not magic. Text."

---

## 1.3 HTTP is Just Text

**Let's prove it. Here's a Python program that sends raw HTTP over a socket.**

> "You don't need to memorize this code. Watch what happens."

```python
# demo_raw_http.py — Proving HTTP is just text
import socket

def send_raw_http() -> None:
    """Send a raw HTTP request using nothing but a socket"""

    # Step 1: Open a connection (just a pipe for bytes)
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.connect(("httpbin.org", 80))

        # Step 2: Write HTTP by hand — it's just a string!
        request_text: str = (
            "GET /get HTTP/1.1\r\n"
            "Host: httpbin.org\r\n"
            "Accept: application/json\r\n"
            "Connection: close\r\n"
            "\r\n"
        )
        
        print("=== WHAT WE SEND (raw text) ===")
        print(request_text)

        sock.sendall(request_text.encode())

        # Step 3: Read what comes back — also just text
        response = b""
        while True:
            chunk = sock.recv(4096)
            if not chunk:
                break
            response += chunk

        print("=== WHAT WE GET BACK (raw text) ===")
        print(response.decode())

send_raw_http()
```

**Run it. Read the output.**

```
=== WHAT WE SEND (raw text) ===
GET /get HTTP/1.1
Host: httpbin.org
Accept: application/json
Connection: close

=== WHAT WE GET BACK (raw text) ===
HTTP/1.1 200 OK
Date: Mon, 09 Feb 2026 10:30:00 GMT
Content-Type: application/json
Content-Length: 256

{
  "args": {},
  "headers": {
    "Accept": "application/json",
    "Host": "httpbin.org"
  },
  "url": "http://httpbin.org/get"
}
```

**The insight:**

> "We typed a plain-text message. We sent it through a network socket. We got a plain-text message back. No magic library. No framework. Just strings through a pipe. That is HTTP."

> "Notice the `with socket...` context manager? That's what you learned in Week 1, Lecture 2. Resource management — open the connection, guarantee it closes."

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE REVELATION                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP is a PROTOCOL — a set of rules for how two                │
│  computers format text messages to each other.                  │
│                                                                 │
│  "Protocol" means: agreed-upon rules for communication.         │
│                                                                 │
│  Like how letters have a format:                                │
│  ├─ Recipient address at the top                                │
│  ├─ Date                                                        │
│  ├─ "Dear [name],"                                              │
│  ├─ Body of the letter                                          │
│  └─ "Sincerely, [name]"                                         │
│                                                                 │
│  HTTP messages have a format too:                               │
│  ├─ Request/Status line at the top                              │
│  ├─ Headers (metadata)                                          │
│  ├─ Empty line (separator)                                      │
│  └─ Body (optional content)                                     │
│                                                                 │
│  HTTP stands for HyperText Transfer Protocol.                   │
│  You'll hear it a thousand times. Now you know what it means.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 The Library Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE LIBRARY ANALOGY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Imagine a LIBRARY with a FRONT DESK.                           │
│                                                                 │
│  You (the visitor) walk up to the front desk.                   │
│  You fill out a REQUEST FORM.                                   │
│  The librarian processes it and hands back a RESPONSE.          │
│                                                                 │
│                                                                 │
│  ┌────────────┐         request form           ┌────────────┐   │
│  │            │ ─────────────────────────────▶ │            │   │
│  │  YOU       │                                │  LIBRARY   │   │
│  │  (Client)  │                                │  (Server)  │   │
│  │            │ ◀───────────────────────────── │            │   │
│  └────────────┘      response + materials      └────────────┘   │
│                                                                 │
│                                                                 │
│  THE RULES:                                                     │
│  ├─ You ALWAYS initiate. The library never walks up to you.     │
│  ├─ You fill out ONE form per request. One request, one reply.  │
│  ├─ Each form has a TYPE: "borrow", "return", "search", etc.    │
│  ├─ The librarian ALWAYS replies, even if the answer is "no".   │
│  └─ The librarian doesn't remember your last visit.             │
│     Every form is self-contained.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to HTTP:**

```
Library Front Desk           │  HTTP
─────────────────────────────│──────────────────────────
You (the visitor)            │  Client (browser, app, script)
The library                  │  Server (your FastAPI app)
Request form                 │  HTTP Request
Librarian's response         │  HTTP Response
Type of form (borrow/return) │  HTTP Method (GET/POST/PUT...)
Form fields (name, card #)   │  Headers (metadata)
What you write on the form   │  Request body (data you send)
What the librarian hands back│  Response body (data you get)
Librarian's stamp (approved, │  Status code (200 OK,
  denied, not found)         │    403 Forbidden, 404 Not Found)
Catalog number / shelf code  │  URL path (/books/42)
Your library card            │  Authorization header
```

**Key rule — the library never calls you first:**

> "HTTP is a CLIENT-INITIATED protocol. The server sits and waits. The client sends a request. The server sends a response. Always in this order. The server NEVER sends a response unprompted."

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ✅ HOW HTTP WORKS:                                             │
│                                                                 │
│     Client  ──── request ────▶  Server                          │
│     Client  ◀─── response ───  Server                           │
│                                                                 │
│  ❌ THIS NEVER HAPPENS IN HTTP:                                 │
│                                                                 │
│     Client  ◀─── "Hey! News!" ──  Server     ← Nope.            │
│                                                                 │
│  (Want the server to push data? That's WebSockets — Week 9.)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 2: ANATOMY OF HTTP

## 2.1 The Request — What You Send

**Every HTTP request has exactly this structure:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    HTTP REQUEST STRUCTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────┐      │
│  │  GET /books/42 HTTP/1.1                ← Request Line │      │
│  ├───────────────────────────────────────────────────────┤      │
│  │  Host: library.example.com             ← Header       │      │
│  │  Accept: application/json              ← Header       │      │
│  │  Authorization: Bearer abc123          ← Header       │      │
│  ├───────────────────────────────────────────────────────┤      │
│  │                                        ← Blank Line   │      │
│  ├───────────────────────────────────────────────────────┤      │
│  │  (no body for GET requests)            ← Body         │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                 │
│  THREE PARTS:                                                   │
│  ─────────────                                                  │
│  1. REQUEST LINE:  [METHOD] [PATH] [HTTP VERSION]               │
│     └─ "What do you want to do, to what, using what version"    │
│                                                                 │
│  2. HEADERS:  Key-value pairs, one per line                     │
│     └─ Metadata about the request (who you are, what you want)  │
│                                                                 │
│  3. BODY:  The actual content (optional, depends on method)     │
│     └─ Separated from headers by ONE blank line                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Dissect the request line:**

```
GET  /books/42  HTTP/1.1
 │       │         │
 │       │         └─── Protocol version (almost always 1.1 for us)
 │       │
 │       └───── PATH: Which resource you want
 │              (the catalog number in our library analogy)
 │
 └──── METHOD: What you want to do with it
        (the type of form you're filling out)
```

**A POST request has a body:**

```
POST /books HTTP/1.1
Host: library.example.com
Content-Type: application/json
Content-Length: 64

{"title": "Clean Code", "author": "Robert Martin", "year": 2008}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  POST /books HTTP/1.1                     ← "I want to ADD      │
│                                              a new book"        │
│  Host: library.example.com                ← "at this library"   │
│  Content-Type: application/json           ← "the data is JSON"  │
│  Content-Length: 64                       ← "it's 64 bytes"     │
│                                           ← blank line          │
│  {"title": "Clean Code", ...}             ← HERE's the book     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 The Response — What Comes Back

**Every HTTP response has the same three-part structure:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    HTTP RESPONSE STRUCTURE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────┐      │
│  │  HTTP/1.1 200 OK                       ← Status Line  │      │
│  ├───────────────────────────────────────────────────────┤      │
│  │  Content-Type: application/json        ← Header       │      │
│  │  Content-Length: 82                    ← Header       │      │
│  │  Date: Mon, 09 Feb 2026 10:30:00 GMT   ← Header       │      │
│  ├───────────────────────────────────────────────────────┤      │
│  │                                        ← Blank Line   │      │
│  ├───────────────────────────────────────────────────────┤      │
│  │  {"id": 42, "title": "Clean Code",     ← Body         │      │
│  │   "author": "Robert Martin",                          │      │
│  │   "available": true}                                  │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                 │
│  THREE PARTS:                                                   │
│  ─────────────                                                  │
│  1. STATUS LINE:  [HTTP VERSION] [STATUS CODE] [REASON PHRASE]  │
│     └─ "Here's what happened with your request"                 │
│                                                                 │
│  2. HEADERS:  Key-value pairs, one per line                     │
│     └─ Metadata about the response                              │
│                                                                 │
│  3. BODY:  The actual content                                   │
│     └─ The data you asked for (or error details)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Dissect the status line:**

```
HTTP/1.1  200  OK
   │       │    │
   │       │    └─── REASON PHRASE: Human-readable label
   │       │         (for your eyes, code ignores this)
   │       │
   │       └──── STATUS CODE: Machine-readable number
   │              (your code checks THIS)
   │
   └──── VERSION: Protocol version
```

**Library analogy: the response is the librarian's reply slip:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  You asked:    "Can I get book #42?"                            │
│                                                                 │
│  Librarian's reply slip:                                        │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  STATUS: ✅ APPROVED (200)                          │        │
│  │  Format: JSON text                                  │        │
│  │  Date processed: Feb 9, 2026                        │        │
│  │                                                     │        │
│  │  Here's your book:                                  │        │
│  │  Title: Clean Code                                  │        │
│  │  Author: Robert Martin                              │        │
│  │  Available: Yes                                     │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  What if the book didn't exist?                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  STATUS: ❌ NOT FOUND (404)                         │        │
│  │  Format: JSON text                                  │        │
│  │                                                     │        │
│  │  Sorry: Book #42 is not in our catalog              │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When Content-Length Is Unknown: Chunked Transfer**

Most responses know their size in advance and send `Content-Length`. But sometimes the server generates data on the fly — a large report, a streaming feed — and cannot know the total size ahead of time.

```
┌─────────────────────────────────────────────────────────────────┐
│              TRANSFER-ENCODING: CHUNKED                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  When Content-Length is unknown, the server sends chunks.       │
│  Each chunk declares its own size (in hexadecimal).             │
│  A zero-length chunk signals the end.                           │
│                                                                 │
│  HTTP/1.1 200 OK                                                │
│  Content-Type: text/plain                                       │
│  Transfer-Encoding: chunked    ← No Content-Length              │
│                                ← blank line                     │
│  4\r\n                         ← Chunk size: 4 bytes (hex)      │
│  Wiki\r\n                      ← Chunk data                     │
│  6\r\n                         ← Next chunk: 6 bytes            │
│  pedia \r\n                    ← Chunk data                     │
│  E\r\n                         ← Next chunk: 14 bytes (E hex)   │
│  in\r\n\r\nchunks.\r\n         ← Chunk data                     │
│  0\r\n                         ← Zero = end of stream           │
│  \r\n                                                           │
│                                                                 │
│  The client reads chunks and assembles the full response body.  │
│                                                                 │
│  You will use this in Week 12 (FastAPI's StreamingResponse).    │
│  Frameworks handle the chunked encoding automatically —         │
│  you never write raw chunk boundaries by hand.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

## 2.3 The Full Cycle (Visual)

**The complete request-response cycle:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE REQUEST-RESPONSE CYCLE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   CLIENT                                          SERVER        │
│   (browser, app,                                  (your         │
│    Python script)                                  FastAPI app) │
│        │                                              │         │
│        │  1. Open connection (TCP)                    │         │
│        │─────────────────────────────────────────────▶│         │
│        │                                              │         │
│        │  2. Send HTTP Request                        │         │
│        │  ┌──────────────────────────────┐            │         │
│        │  │ GET /users/42 HTTP/1.1       │            │         │
│        │  │ Host: api.example.com        │───────────▶│         │
│        │  │ Accept: application/json     │            │         │
│        │  └──────────────────────────────┘            │         │
│        │                                              │         │
│        │              3. Server processes request     │         │
│        │                 (find user #42 in database)  │         │
│        │                                              │         │
│        │  4. Send HTTP Response                       │         │
│        │  ┌──────────────────────────────┐            │         │
│        │  │ HTTP/1.1 200 OK              │            │         │
│        │  │Content-Type: application/json│◀───────────│         │
│        │  │                              │            │         │
│        │  │ {"id":42, "name":"Alice"}    │            │         │
│        │  └──────────────────────────────┘            │         │
│        │                                              │         │
│        │  5. Close connection                         │         │
│        │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ - - │         │
│        │                                              │         │
│                                                                 │
│  ONE request → ONE response. Always. No exceptions.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
**Connection Lifecycle — Persistent Connections**

> "Notice Step 1 says 'Open connection (TCP)' and Step 5 says 'Close connection'. In early HTTP, this happened for every single request — open, request, response, close. For a page with 30 images, that's 30 TCP handshakes. Slow."

```
┌─────────────────────────────────────────────────────────────────┐
│         PERSISTENT CONNECTIONS (HTTP/1.1 DEFAULT)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP/1.0 — One request, one connection:                        │
│                                                                 │
│  Client ──open──▶ Server                                        │
│  Client ──req 1──▶ Server                                       │
│  Client ◀──res 1── Server                                       │
│  Client ──CLOSE── Server   ← connection discarded               │
│  Client ──open──▶ Server   ← new handshake for req 2            │
│  ...repeated for every asset...                                 │
│                                                                 │
│                                                                 │
│  HTTP/1.1 — Keep-alive (persistent, default):                   │
│                                                                 │
│  Client ──open──────────────────────────────────▶ Server        │
│  Client ──req 1─────────────────────────────────▶ Server        │
│  Client ◀──res 1──────────────────────────────── Server         │
│  Client ──req 2─────────────────────────────────▶ Server        │
│  Client ◀──res 2──────────────────────────────── Server         │
│  Client ──CLOSE──────────────────────────────── Server          │
│          ↑ same connection used for all requests                │
│                                                                 │
│  Connection: keep-alive  ← HTTP/1.1 default (usually implicit)  │
│  Connection: close       ← "Close after this one request"       │
│                                                                 │
│  That's why the raw socket demo in 1.3 sent Connection: close   │
│  — without it, the server keeps the connection open and the     │
│  recv() loop hangs waiting for data that never arrives.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection pooling in API clients:**

```python
import httpx

# BAD — new TCP connection for every request (100 handshakes)
for book_id in range(100):
    response = httpx.get(f"https://api.library.com/books/{book_id}")
    # Each call opens and immediately closes a TCP connection

# GOOD — shared connection pool (far fewer handshakes)
async with httpx.AsyncClient() as client:
    for book_id in range(100):
        response = await client.get(f"https://api.library.com/books/{book_id}")
        # AsyncClient reuses connections from its internal pool
```

> "One TCP connection, many requests — this is exactly why async shines. While the client awaits one response, it can fire the next request on the same underlying connection. Week 8 covers `httpx.AsyncClient` connection pool configuration in full."

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏️  PRACTICE CHECKPOINT 2.3                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q1. The raw socket demo in 1.3 sent "Connection: close".       │
│      What would happen if that header were omitted?             │
│                                                                 │
│  Q2. Your script makes 500 API calls to an external service.    │
│      Should you use a new httpx.get() per call or a shared      │
│      httpx.AsyncClient()? State one concrete reason.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
```
┌─────────────────────────────────────────────────────────────────┐
│                   ✅  CHECKPOINT SOLUTIONS 2.3                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A1. The server keeps the connection open (keep-alive). The     │
│      recv() loop in the demo has no way to detect "headers      │
│      done" vs "body done" on a persistent connection — it       │
│      blocks forever waiting for bytes that never come, until    │
│      the server-side idle timeout closes the connection.        │
│                                                                 │
│  A2. Use httpx.AsyncClient(). Each standalone httpx.get() opens │
│      and tears down a TCP connection — that's 500 handshakes.   │
│      AsyncClient maintains a connection pool and reuses         │
│      connections, eliminating redundant handshake overhead.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
**Now ask the class:**

> "During Step 3 — the server processing the request — what is the client doing?"

Answer: **Waiting.** Blocked. Doing nothing.

> "Sound familiar? This is exactly the I/O-bound waiting from Lecture 3 last week. The client sends a request and WAITS for the response. That's why you learned async. When your FastAPI server handles hundreds of requests, async lets it process other requests while waiting for database queries — not one at a time."

---

## 2.4 Seeing It Live (curl and Python)

**curl is the backend developer's best friend.**

`curl` (short for "Client URL") is a command-line tool for making HTTP requests. You will use it every single day.

```bash
# Basic GET request
curl https://httpbin.org/get
```

```bash
# -v (verbose) shows the FULL conversation — request AND response
curl -v https://httpbin.org/get
```

**The verbose output shows everything:**

```
* Connecting to httpbin.org port 443
> GET /get HTTP/1.1              ← YOUR REQUEST starts here
> Host: httpbin.org
> User-Agent: curl/8.1.2
> Accept: */*
>                                ← Blank line (end of request)
< HTTP/1.1 200 OK               ← SERVER RESPONSE starts here
< Content-Type: application/json
< Content-Length: 256
<                                ← Blank line (end of headers)
{                                ← Response body
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.org"
  },
  "url": "https://httpbin.org/get"
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   READING CURL -v OUTPUT                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Lines starting with  >  are what YOU SENT (request)            │
│  Lines starting with  <  are what the SERVER SENT (response)    │
│  Lines starting with  *  are curl's own notes                   │
│                                                                 │
│  Get comfortable with this output format.                       │
│  You will read it hundreds of times while debugging.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Sending different types of requests with curl:**

```bash
# GET request (default)
curl https://httpbin.org/get

# POST request with JSON body
curl -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "age": 30}'

# PUT request
curl -X PUT https://httpbin.org/put \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "age": 31}'

# DELETE request
curl -X DELETE https://httpbin.org/delete
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   CURL FLAGS CHEAT SHEET                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  -v               Verbose (show full conversation)              │
│  -X METHOD        Specify HTTP method (POST, PUT, DELETE...)    │
│  -H "Key: Value"  Add a header                                  │
│  -d '...'         Send request body (data)                      │
│  -i               Show response headers                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
**The Browser DevTools Network Tab**

> "Before you reach for curl, check your browser. The DevTools Network tab shows every HTTP conversation your browser has — in real time, full headers, request and response bodies. It is the most immediately accessible debugging tool you have."

**Opening DevTools:**

```
Chrome / Edge / Brave:    F12  (or Ctrl+Shift+I / Cmd+Option+I on Mac)
Firefox:                  F12  (or Ctrl+Shift+I)
Safari:                   Cmd+Option+I  (enable developer menu first)

Then: click the "Network" tab.
Reload the page if no requests are visible.
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   DEVTOOLS NETWORK TAB                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request list:                                                  │
│  Method  Status  Type    Name               Time               │
│  ──────  ──────  ──────  ─────────────────  ──────             │
│  GET     200     fetch   books              48ms               │
│  POST    201     fetch   books              121ms              │
│  GET     404     fetch   books/9999         32ms               │
│                                                                 │
│  Click any row → four inner tabs appear:                        │
│                                                                 │
│  Headers   ← full request headers + full response headers       │
│  Preview   ← parsed response (JSON tree, image preview, etc.)   │
│  Response  ← raw response body text                             │
│  Timing    ← DNS lookup, TCP connect, wait, download breakdown  │
│                                                                 │
│  Filter bar:                                                    │
│  All  Fetch/XHR  JS  CSS  Img  Media  Font  Doc  WS  Other     │
│       ↑                                                         │
│       Click "Fetch/XHR" to show ONLY API calls.                 │
│       This removes the noise of JS/CSS/image loading.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The "Copy as cURL" bridge — your most useful debugging move:**

```
Right-click any request in the list
  → "Copy"
  → "Copy as cURL"
```

Example output pasted into your terminal:

```bash
curl 'https://api.library.com/books/42' \
  -H 'Accept: application/json' \
  -H 'Authorization: Bearer eyJhbG...' \
  -H 'Content-Type: application/json' \
  --compressed
```

> "This is the bridge between browser debugging and terminal debugging. A request fails in your React app? Right-click → Copy as cURL → run it in your terminal. Now you can modify headers, change the method, strip the auth token, and experiment — without touching frontend code. This workflow will save you hours."

> "You will spend more time in this tab than in any other part of your browser. Learn to read it fluently."

**Now, the Python version — using only what you already know:**

```python
# demo_http_client.py
# Using Python's built-in http.client (no pip install needed)
import http.client
import json

def fetch_book(book_id: int) -> dict:
    """Make a GET request using the standard library"""

    # Open a connection (context manager — Week 1, Lecture 2)
    connection = http.client.HTTPSConnection("httpbin.org")

    try:
        # Send the request
        connection.request("GET", f"/get?book_id={book_id}")

        # Read the response
        response = connection.getresponse()

        print(f"Status: {response.status} {response.reason}")
        print(f"Headers: {dict(response.getheaders())}")

        body: str = response.read().decode()
        data: dict = json.loads(body)
        print(f"Body: {json.dumps(data, indent=2)}")

        return data
    finally:
        connection.close()

result = fetch_book(42)
```

> "Libraries like `requests` or `httpx` (you'll learn in Week 4) wrap all this into cleaner APIs. But underneath? They're doing exactly what you see here — opening a connection, writing text, reading text back."

---
## 2.5 The HTTP Evolution — HTTP/1.1 → HTTP/2 → HTTP/3

> "Everything in this lecture is HTTP/1.1. It is text-based, human-readable, and has been the web's backbone since 1997. But HTTP did not stop there. You need to know what came after — not to implement it, but so that 'multiplexing', 'binary framing', and 'QUIC' don't sound like alien words when you hear them in a production context."

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE HTTP EVOLUTION                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP/1.1  (1997 — still the majority of the internet)          │
│  ──────────────────────────────────────────────────────         │
│  ├─ Plain text — human-readable, as you have been reading       │
│  ├─ One outstanding request per connection at a time            │
│     (head-of-line blocking)                                     │
│  ├─ Keep-alive allows reuse, but requests are still sequential  │
│  └─ Headers are repeated in full on every single request         │
│     (no compression — wasteful for large header sets)           │
│                                                                 │
│                                                                 │
│  HTTP/2  (2015 — widely supported, most modern APIs use this)   │
│  ────────────────────────────────────────────────────────────   │
│  ├─ BINARY — not text. Faster to parse, more compact            │
│  ├─ MULTIPLEXING: multiple requests travel simultaneously over  │
│  │   ONE TCP connection (no head-of-line blocking)              │
│  ├─ HEADER COMPRESSION (HPACK): headers are compressed and      │
│  │   cached — repeated headers sent only once                   │
│  └─ Server push (server pre-sends resources the client          │
│     will likely need — rarely used in practice)                 │
│                                                                 │
│  Key gain: a page with 100 assets no longer needs 100 TCP       │
│  connections or 100 sequential round-trips.                     │
│                                                                 │
│                                                                 │
│  HTTP/3  (2022 — growing adoption, ~30% of web traffic)         │
│  ────────────────────────────────────────────────────────────   │
│  ├─ Runs over QUIC (UDP-based transport) instead of TCP         │
│  ├─ 0-RTT connection setup for returning clients (faster)       │
│  ├─ Per-stream error recovery — one lost packet stalls only     │
│  │   its own stream, not the entire connection (fixes the one   │
│  │   remaining flaw in HTTP/2's TCP foundation)                 │
│  └─ TLS 1.3 encryption is mandatory — no plain HTTP/3           │
│                                                                 │
│                                                                 │
│  THE KEY INSIGHT — SEMANTICS NEVER CHANGE:                      │
│  ──────────────────────────────────────────                     │
│  GET, POST, DELETE, PUT, PATCH                                  │
│  200 OK, 404 Not Found, 422 Unprocessable                       │
│  Content-Type, Authorization, Cache-Control                     │
│                                                                 │
│  Identical across HTTP/1.1, HTTP/2, and HTTP/3.                 │
│  Only the TRANSPORT layer changes. The vocabulary you are       │
│  learning today works on all three versions — forever.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Head-of-line blocking: the problem HTTP/2 solves:**

```
┌─────────────────────────────────────────────────────────────────┐
│           HEAD-OF-LINE BLOCKING VISUALISED                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP/1.1 (sequential — one request must finish before next):   │
│                                                                 │
│  ──[Req A]──────[Res A]──[Req B]──────[Res B]──[Req C]──▶       │
│                          ↑ Req B cannot start until Res A done  │
│                                                                 │
│                                                                 │
│  HTTP/2 (multiplexed — concurrent over ONE connection):         │
│                                                                 │
│  Stream 1:  ──[Req A]──────────────[Res A]──────────────▶       │
│  Stream 2:     ──[Req B]──[Res B]──────────────────────▶        │
│  Stream 3:        ──[Req C]────────────[Res C]─────────▶        │
│                                                                 │
│  All three requests in flight simultaneously.                   │
│  No waiting. One TCP connection.                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What this means for your FastAPI code:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Your local dev server:                                         │
│  http://localhost:8000   → HTTP/1.1 (uvicorn default)           │
│                                                                 │
│  Your production FastAPI:                                       │
│  https://api.example.com → HTTP/2 (via Nginx or cloud LB)       │
│                                                                 │
│  Your FastAPI application code changes: zero.                   │
│  uvicorn speaks HTTP. Nginx (or your cloud platform) handles    │
│  the HTTP version upgrade and TLS. You do not implement this.   │
│  Week 15 covers the deployment plumbing.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏️  PRACTICE CHECKPOINT 2.5                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q1. What is head-of-line blocking, and which HTTP version      │
│      eliminates it at the application layer?                    │
│                                                                 │
│  Q2. A teammate says: "Our production API is HTTP/2, so your    │
│      knowledge of GET, POST, and status codes is outdated."     │
│      Are they correct? Justify your answer.                     │
│                                                                 │
│  Q3. HTTP/3 runs over UDP rather than TCP. In HTTP/2 over TCP,  │
│      a single lost packet stalls the entire connection.         │
│      How does HTTP/3 over QUIC solve this?                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
```
┌─────────────────────────────────────────────────────────────────┐
│                   ✅  CHECKPOINT SOLUTIONS 2.5                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A1. Head-of-line blocking: in HTTP/1.1, request B cannot be    │
│      sent until the response to request A is fully received.    │
│      HTTP/2 eliminates this at the HTTP layer by multiplexing   │
│      concurrent streams over one TCP connection.                │
│      (TCP itself still has its own head-of-line blocking,        │
│      which HTTP/3/QUIC eliminates at the transport layer.)      │
│                                                                 │
│  A2. No. HTTP semantics — methods, status codes, headers, URL   │
│      conventions — are identical across all HTTP versions.      │
│      HTTP/2 changes how messages are encoded and transported;   │
│      it does not change what the messages mean. Everything      │
│      taught in this lecture applies equally to HTTP/2.          │
│                                                                 │
│  A3. QUIC implements independent streams at the transport layer. │
│      A lost UDP packet only affects the single stream it         │
│      belongs to. The other streams continue uninterrupted.      │
│      In TCP, the entire connection stalls because TCP's          │
│      reliability guarantees are per-connection, not per-stream. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---
# PART 3: HTTP METHODS — THE FIVE VERBS

## 3.1 Why Multiple Methods?

**Why can't everything just be GET?**

> "At the library, you don't use the same form to borrow a book, donate a book, and cancel your membership. Different actions require different forms. HTTP is the same."

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHY MULTIPLE METHODS?                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Imagine a library with only ONE type of form:                  │
│                                                                 │
│  FORM:  "Do something with book #42"                            │
│                                                                 │
│  Librarian: "Do WHAT? Borrow it? Return it? Burn it?"           │
│                                                                 │
│  HTTP methods TELL the server what kind of operation            │
│  you want to perform. The server doesn't have to guess.         │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │                                                     │        │
│  │  GET     →  "Give me this resource"                 │        │
│  │  POST    →  "Create a new resource"                 │        │
│  │  PUT     →  "Replace this entire resource"          │        │
│  │  PATCH   →  "Update part of this resource"          │        │
│  │  DELETE  →  "Remove this resource"                  │        │
│  │                                                     │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  The method is a SEMANTIC SIGNAL. It tells both the server      │
│  AND anyone reading the code what the INTENTION is.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 GET — Retrieve a Resource

**"Show me this book."**

```
┌─────────────────────────────────────────────────────────────────┐
│                          GET                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PURPOSE:  Retrieve data. Read. Look, don't touch.              │
│                                                                 │
│  BODY:     ❌ No request body (you're asking, not sending)      │
│                                                                 │
│  LIBRARY:  "I'd like to see the record for book #42."           │
│            You hand the librarian a slip with the catalog       │
│            number. They hand you back the book info.            │
│            The book doesn't change. The library doesn't change. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
GET /books/42 HTTP/1.1
Host: library.example.com
Accept: application/json
```

```
HTTP/1.1 200 OK
Content-Type: application/json

{"id": 42, "title": "Clean Code", "author": "Robert Martin"}
```

**GET is the most common method.** Every time you type a URL in your browser, every time a page loads an image — that's a GET.

**GET can include query parameters — extra filters on your request:**

```
GET /books?author=martin&available=true HTTP/1.1
Host: library.example.com
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    URL WITH QUERY PARAMS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  https://library.example.com/books?author=martin&available=true │
│  ──┬──   ───────┬──────────  ──┬── ─────────────┬────────────   │
│    │            │              │                 │              │
│  scheme       host           path         query parameters      │
│                                                                 │
│  Query parameters:                                              │
│  ├─ Start after the ?                                           │
│  ├─ key=value pairs                                             │
│  └─ Separated by &                                              │
│                                                                 │
│  Library analogy: "Show me books WHERE author is Martin         │
│                    AND available is true."                      │
│                    You're filtering, not changing anything.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 POST — Create a Resource

**"I'd like to donate a new book."**

```
┌─────────────────────────────────────────────────────────────────┐
│                          POST                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PURPOSE:  Create something new.                                │
│                                                                 │
│  BODY:     ✅ Yes — the data for the new resource               │
│                                                                 │
│  LIBRARY:  "I'm donating a new book. Here's all the info."      │
│            You fill out a donation form with the book's title,  │
│            author, ISBN. The librarian adds it to the catalog   │
│            and hands you back a confirmation with the new       │
│            catalog number.                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
POST /books HTTP/1.1
Host: library.example.com
Content-Type: application/json

{"title": "Design Patterns", "author": "Gang of Four", "year": 1994}
```

```
HTTP/1.1 201 Created
Content-Type: application/json
Location: /books/99

{"id": 99, "title": "Design Patterns", "author": "Gang of Four", "year": 1994}
```

**Notice:**
- The request path is `/books` (the collection), not `/books/99`
- The server ASSIGNS the ID (99) — you don't choose it
- The status code is `201 Created`, not `200 OK`
- The `Location` header tells you WHERE the new resource lives

---

## 3.4 PUT vs PATCH — Replace vs Update

**This distinction confuses everyone. Let's make it clear.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      PUT vs PATCH                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PUT = FULL REPLACEMENT                                         │
│  ──────────────────────                                         │
│  "Replace the ENTIRE record with this new one."                 │
│                                                                 │
│  Library: "Here is a completely new record for book #42.        │
│           Throw away everything you had before."                │
│                                                                 │
│  If you forget a field, it's GONE.                              │
│                                                                 │
│                                                                 │
│  PATCH = PARTIAL UPDATE                                         │
│  ──────────────────────                                         │
│  "Just change THESE specific fields. Keep everything else."     │
│                                                                 │
│  Library: "For book #42, just change the availability           │
│           to 'checked out'. Don't touch anything else."         │
│                                                                 │
│  Only the fields you send are modified.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**See the difference:**

```
Current state of book #42:
{"id": 42, "title": "Clean Code", "author": "Robert Martin", "available": true}
```

**PUT /books/42 — full replacement:**

```
PUT /books/42 HTTP/1.1
Content-Type: application/json

{"id": 42, "title": "Clean Code", "author": "Robert C. Martin", "available": true}
```

You must send EVERY field. If you send this instead:

```json
{"id": 42, "author": "Robert C. Martin"}
```

The result would be: `{"id": 42, "author": "Robert C. Martin"}` — title and available are **gone**. You replaced the entire record.

**PATCH /books/42 — partial update:**

```
PATCH /books/42 HTTP/1.1
Content-Type: application/json

{"available": false}
```

Result: `{"id": 42, "title": "Clean Code", "author": "Robert Martin", "available": false}`

Only `available` changed. Everything else stayed.

```
┌─────────────────────────────────────────────────────────────────┐
│                   PUT vs PATCH VISUALIZED                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE:  { title: "Clean Code",                                │
│             author: "Robert Martin",                            │
│             available: true }                                   │
│                                                                 │
│                                                                 │
│  PUT {author: "Robert C. Martin"}                               │
│                                                                 │
│  AFTER:   { author: "Robert C. Martin" }                        │
│           ⚠️ title and available are GONE                       │
│           PUT = "replace the whole thing with what I gave you"  │
│                                                                 │
│                                                                 │
│  PATCH {author: "Robert C. Martin"}                             │
│                                                                 │
│  AFTER:   { title: "Clean Code",                                │
│             author: "Robert C. Martin",    ← changed            │
│             available: true }              ← kept               │
│           PATCH = "only change what I specified"                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "In practice, most APIs use PATCH for updates because it's safer — you can't accidentally erase fields. You'll see both, but PATCH is more common in modern APIs."

---

## 3.5 DELETE — Remove a Resource

**"Remove this book from the catalog."**

```
┌─────────────────────────────────────────────────────────────────┐
│                         DELETE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PURPOSE:  Remove a resource.                                   │
│                                                                 │
│  BODY:     ❌ Usually no request body                           │
│                                                                 │
│  LIBRARY:  "Please remove book #42 from the catalog."           │
│            The librarian confirms: "Done." (or "That book       │
│            didn't exist anyway.")                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
DELETE /books/42 HTTP/1.1
Host: library.example.com
Authorization: Bearer admin-token-here
```

```
HTTP/1.1 204 No Content
```

**Notice:**

- `204 No Content` — success, but nothing to return (the resource is gone)
- No response body — there's nothing to show you
- Usually requires authorization — you don't let just anyone delete things

---

## 3.6 Safety, Idempotency, and Idempotency Keys

**Two formal properties that MATTER for API design:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  SAFE = Does NOT change anything on the server                  │
│  ────                                                           │
│  Reading a book doesn't remove it from the shelf.               │
│  You can look at it 100 times; the library is unchanged.        │
│                                                                 │
│                                                                 │
│  IDEMPOTENT = Doing it once or 100 times gives the SAME result  │
│  ──────────                                                     │
│  "Set book #42's status to available." Do it 1 time or 50 —     │
│  the result is the same: book #42 is available.                 │
│                                                                 │
│  Contrast with: "Add a new book." Do it 50 times and you        │
│  get 50 duplicate books. That's NOT idempotent.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│              METHOD PROPERTIES CHEAT SHEET                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Method    Safe?     Idempotent?   Has Body?                    │
│  ───────   ──────    ───────────   ─────────                    │
│  GET       ✅ Yes    ✅ Yes        ❌ No                         │
│  POST      ❌ No     ❌ No         ✅ Yes                        │
│  PUT       ❌ No     ✅ Yes        ✅ Yes                        │
│  PATCH     ❌ No     ❌ No*        ✅ Yes                        │
│  DELETE    ❌ No     ✅ Yes        ❌ Usually no                 │
│                                                                 │
│  * PATCH is not guaranteed idempotent — depends on              │
│    implementation. (e.g., "increment count by 1" is not         │
│    idempotent, but "set count to 5" is.)                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why does idempotency matter?**

> "Imagine a user clicks 'Submit Order' and the network is slow. They click again. And again. If your POST endpoint isn't designed for this, you just created 3 orders. Idempotency is how you think about these real-world problems."

```
┌─────────────────────────────────────────────────────────────────┐
│             WHY IDEMPOTENCY MATTERS IN THE REAL WORLD           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IDEMPOTENT (safe to retry):                                    │
│                                                                 │
│  PUT /books/42  {"available": false}    ← Do it 3 times?        │
│  PUT /books/42  {"available": false}      Book is unavailable.  │
│  PUT /books/42  {"available": false}      Same result. Fine.    │
│                                                                 │
│                                                                 │
│  NOT IDEMPOTENT (dangerous to retry):                           │
│                                                                 │
│  POST /orders   {"item": "book", ...}  ← Do it 3 times?         │
│  POST /orders   {"item": "book", ...}    3 SEPARATE orders!     │
│  POST /orders   {"item": "book", ...}    Customer charged 3x!   │
│                                                                 │
│                                                                 │
│  This is why GET, PUT, and DELETE are designed as idempotent    │
│  — network failures happen, and retries must be safe.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
**The Solution: Idempotency Keys**

> "We just established that POST is not idempotent and that retrying a failed POST can duplicate orders and charges. So what is the industry solution? The **idempotency key** — a unique ID that the client generates and attaches to the request. The server uses it to detect duplicates and return a cached result instead of processing again."

```
┌─────────────────────────────────────────────────────────────────┐
│                    IDEMPOTENCY KEYS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HOW IT WORKS:                                                  │
│                                                                 │
│  1. Client generates a UUID for this specific operation         │
│                                                                 │
│  2. Client sends it as a header:                                │
│     Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000       │
│                                                                 │
│  3. FIRST request with that key:                                │
│     Server processes normally, stores the result keyed by UUID  │
│                                                                 │
│  4. DUPLICATE request (same key, network retry):                │
│     Server recognises the key, returns the stored result        │
│     — no second charge, no second order, no second email        │
│                                                                 │
│  5. After a timeout (e.g. 24 hours), the stored result expires  │
│     The key may then be reused for a genuinely new operation    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The Stripe pattern — the most famous real-world example:**

```python
import uuid
import httpx

# The client generates one stable key for this payment attempt.
# If the network drops and we retry, we use THE SAME key.
idempotency_key = str(uuid.uuid4())

async def charge_customer_safely(customer_id: str, amount_cents: int) -> dict:
    """
    Charge a customer. Safe to retry — the key prevents double-charging.
    """
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "https://api.stripe.com/v1/charges",
            headers={
                "Idempotency-Key": idempotency_key,  # Same key on every retry
                "Authorization": "Bearer sk_test_...",
            },
            json={
                "customer": customer_id,
                "amount": amount_cents,
                "currency": "usd",
            },
        )
        return response.json()

# Network drops. You call charge_customer_safely() again.
# Server sees the same Idempotency-Key.
# Returns the original result. Customer is charged exactly once.
```

```
┌─────────────────────────────────────────────────────────────────┐
│             IDEMPOTENCY KEY — REQUEST FLOW                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FIRST REQUEST:                                                 │
│  POST /orders                                                   │
│  Idempotency-Key: abc-123      ← client-generated UUID          │
│  {"item": "book", "qty": 1}                                     │
│                                                                 │
│  Server: "First time I've seen abc-123. Processing."            │
│  → Creates order #99                                            │
│  → Stores: {key: "abc-123", result: {"order_id": 99}}           │
│  ← 201 Created: {"order_id": 99}                                │
│                                                                 │
│                                                                 │
│  NETWORK DROPPED. CLIENT RETRIES:                               │
│  POST /orders                                                   │
│  Idempotency-Key: abc-123      ← SAME key                       │
│  {"item": "book", "qty": 1}                                     │
│                                                                 │
│  Server: "Seen abc-123 before. Returning stored result."        │
│  → No new order created                                         │
│  ← 200 OK: {"order_id": 99}    ← same result                    │
│                                                                 │
│  Result: One order exists. One charge. Client has its answer.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "You will implement idempotency key handling on the **server side** in Week 4 (API Design Principles). Storage is typically Redis — fast lookup, automatic TTL expiry. For now, understand the concept and recognise the header when you see it."

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏️  PRACTICE CHECKPOINT 3.6                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q1. DELETE /orders/42 is called, the response is lost in       │
│      transit, and the client retries. The order no longer       │
│      exists. Should the server return 404 or 204? Why?          │
│                                                                 │
│  Q2. A client retries a POST /payments with the SAME            │
│      Idempotency-Key but a DIFFERENT body (changed the amount). │
│      What should the server do?                                 │
│                                                                 │
│  Q3. True or False: PUT /items/99 {"price": 25} executed three  │
│      times leaves the price at 25 after all three calls.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
```
┌─────────────────────────────────────────────────────────────────┐
│                   ✅  CHECKPOINT SOLUTIONS 3.6                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A1. 204 No Content is the more defensible choice. DELETE is    │
│      idempotent: the intended outcome (the order no longer      │
│      exists) is the same whether it was deleted on the first    │
│      or the second call. Returning 404 on a retry breaks        │
│      clients that treat non-2xx as a failure. Both are argued   │
│      in practice; 204 is the cleaner idempotent contract.       │
│                                                                 │
│  A2. Return 409 Conflict or 422 Unprocessable Entity. The key   │
│      was already used for a different request body. The server  │
│      cannot know which body represents the true intent.         │
│      Reject and require the client to generate a new key.       │
│                                                                 │
│  A3. True. PUT is idempotent. Applying "set price to 25" once   │
│      or a hundred times yields the same final state: price=25.  │
│      Contrast with a PATCH {"price_delta": +5} which would      │
│      increment three times — that PATCH is NOT idempotent.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: STATUS CODES — THE SERVER'S VOCABULARY

## 4.1 The Five Families

**Status codes are three-digit numbers. The FIRST digit tells you the category.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE FIVE FAMILIES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1xx — INFORMATIONAL  "Hold on, I'm working on it..."           │
│         (Rare. You almost never deal with these.)               │
│                                                                 │
│  2xx — SUCCESS        "Here you go. It worked." ✅              │
│         (The happy path.)                                       │
│                                                                 │
│  3xx — REDIRECTION    "It's not here. Go look over there." ↪️   │
│         (The resource moved.)                                   │
│                                                                 │
│  4xx — CLIENT ERROR   "YOU did something wrong." ❌             │
│         (Bad request, not found, unauthorized.)                 │
│                                                                 │
│  5xx — SERVER ERROR   "WE did something wrong." 💥              │
│         (Server crashed, overloaded, misconfigured.)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The library version:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  2xx:  Librarian hands you the book with a smile.               │
│        "Here's what you asked for."                             │
│                                                                 │
│  3xx:  "That book was moved to the branch on Oak Street.        │
│         Go there instead."                                      │
│                                                                 │
│  4xx:  "I can't help you because..."                            │
│        ├─ "...you didn't fill out the form right." (400)        │
│        ├─ "...you don't have a library card." (401)             │
│        ├─ "...your card doesn't allow access here." (403)       │
│        └─ "...that book doesn't exist." (404)                   │
│                                                                 │
│  5xx:  "I'm sorry, our systems are down."                       │
│        ├─ "The catalog system crashed." (500)                   │
│        └─ "We're overwhelmed, come back later." (503)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 2xx — It Worked

```
┌─────────────────────────────────────────────────────────────────┐
│                    SUCCESS CODES (2xx)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  200 OK                                                         │
│  ──────                                                         │
│  The standard success response.                                 │
│  "Here's exactly what you asked for."                           │
│  Used with: GET (returning data), PUT/PATCH (returning updated) │
│                                                                 │
│  201 Created                                                    │
│  ───────────                                                    │
│  A new resource was successfully created.                       │
│  "Your new book has been added to the catalog."                 │
│  Used with: POST (new resource created)                         │
│  Often includes a Location header pointing to the new resource. │
│                                                                 │
│  204 No Content                                                 │
│  ──────────────                                                 │
│  Success, but nothing to return.                                │
│  "Done. Book deleted. There's nothing to show you."             │
│  Used with: DELETE (resource removed), PUT (update, no return)  │
│  ⚠️ The response body MUST be empty with 204.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When to use which:**

```python
# As a BACKEND DEVELOPER, you'll decide which status code to return.
# This is a preview of what you'll do in FastAPI:

# GET /books/42 → found the book
#   Return: 200 OK + the book data

# POST /books → created a new book
#   Return: 201 Created + the new book data + Location header

# DELETE /books/42 → deleted the book
#   Return: 204 No Content (empty body)

# PUT /books/42 → updated the whole book record
#   Return: 200 OK + the updated book data
```

---

## 4.3 3xx — Look Elsewhere (Brief)

**The four redirect codes you need to know:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  REDIRECTION CODES (3xx)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────┬──────────────────┬─────────────┬─────────────────┐    │
│  │ Code │ Name             │ Permanent?  │ Method Kept?    │    │
│  ├──────┼──────────────────┼─────────────┼─────────────────┤    │
│  │ 301  │ Moved Permanently│  ✅ Yes     │ ❌ No (→ GET)   │    │
│  │ 302  │ Found (Temp.)    │  ❌ No      │ ❌ No (→ GET)   │    │
│  │ 307  │ Temp. Redirect   │  ❌ No      │ ✅ Yes          │    │
│  │ 308  │ Perm. Redirect   │  ✅ Yes     │ ✅ Yes          │    │
│  └──────┴──────────────────┴─────────────┴─────────────────┘    │
│                                                                 │
│  "Method Kept?" = After redirect, does a POST stay a POST?      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The critical trap — method downgrading in 301/302:**

```
┌─────────────────────────────────────────────────────────────────┐
│              301/302 vs 307/308 — METHOD TRAP                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT POSTs to a URL that has moved:                          │
│                                                                 │
│  With 301 or 302 (method NOT preserved):                        │
│  Client:  POST /old-endpoint  {"data": "..."}                   │
│  Server:  301  Location: /new-endpoint                          │
│  Browser: GET  /new-endpoint      ← downgraded to GET!          │
│                                     request BODY IS LOST        │
│                                                                 │
│  This behaviour violates the RFC but all browsers do it.        │
│  The spec accepted it as a historical reality.                  │
│                                                                 │
│  With 307 or 308 (method IS preserved):                         │
│  Client:  POST /old-endpoint  {"data": "..."}                   │
│  Server:  307  Location: /new-endpoint                          │
│  Client:  POST /new-endpoint  {"data": "..."} ← same method     │
│                                                 + body intact   │
│                                                                 │
│  RULE: Redirecting GET-only resources? 301 or 302.              │
│        Redirecting API endpoints (POST/PUT/PATCH)? 307 or 308.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Each code explained in full:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  301 Moved Permanently                                          │
│  ─────────────────────                                          │
│  "This URL has permanently moved. Update your bookmarks."       │
│  Browsers and search engines update to the new URL.             │
│  ⚠️ Method may be changed to GET. Do NOT use for API            │
│     endpoints that accept POST/PUT/DELETE bodies.               │
│                                                                 │
│  302 Found  (Temporary Redirect)                                │
│  ──────────────────────────────                                 │
│  "Temporarily elsewhere. Keep using this URL next time."        │
│  Clients do NOT cache the redirect permanently.                 │
│  ⚠️ Method may also be changed to GET.                          │
│  Common web use: POST /login → success → 302 → GET /dashboard   │
│  (The POST/Redirect/GET pattern — method downgrade is desired.) │
│                                                                 │
│  307 Temporary Redirect                                         │
│  ──────────────────────                                         │
│  "Temporarily at a new URL. Use the same method."               │
│  POST stays POST. PUT stays PUT. Body is preserved.             │
│  Use when temporarily rerouting API traffic.                    │
│                                                                 │
│  308 Permanent Redirect                                         │
│  ──────────────────────                                         │
│  "Permanently moved. Use the same method."                      │
│  The correct modern choice for API versioning migrations.       │
│  POST /v1/orders → 308 → POST /v2/orders  (body intact)         │
│  Clients must update their URL, but method and body survive.    │
│                                                                 │
│                                                                 │
│  304 Not Modified  (Conditional Requests — see 5.4)             │
│  ──────────────────                                             │
│  "Nothing changed since your cached copy. Use it."              │
│  Response body is EMPTY — the client reuses its local cache.    │
│  Triggered by ETag / If-None-Match headers.                     │
│  This is not really a "redirect" — it is a cache validation     │
│  response. It lives in 3xx for historical reasons.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏️  PRACTICE CHECKPOINT 4.3                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q1. You are permanently moving /v1/orders to /v2/orders.       │
│      Clients POST to /v1/orders with a JSON body.               │
│      Which status code should you return, and why?              │
│                                                                 │
│  Q2. After a successful login form POST, you want the browser   │
│      to navigate to /dashboard with a GET. Which code?          │
│                                                                 │
│  Q3. A developer uses 301 to permanently redirect               │
│      POST /v1/payments to POST /v2/payments.                    │
│      Describe the bug that will result.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

---

┌─────────────────────────────────────────────────────────────────┐
│                   ✅  CHECKPOINT SOLUTIONS 4.3                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A1. 308 Permanent Redirect. The move is permanent (not 307),   │
│      and you need the POST method and request body preserved    │
│      (not 301) so the JSON payload survives the redirect.       │
│                                                                 │
│  A2. 302 Found. You WANT the browser to downgrade POST → GET    │
│      so it performs a clean GET on /dashboard. The method-      │
│      downgrading behaviour of 302 is the desired feature here.  │
│      This is the Post/Redirect/Get (PRG) web pattern.           │
│                                                                 │
│  A3. Browsers downgrade POST to GET on a 301. The redirect      │
│      fires as GET /v2/payments with no body. The payment        │
│      data is gone. The endpoint receives an empty request,      │
│      returns 422 or 400, and no payment is created. The client  │
│      may interpret this as a silent failure or crash on the     │
│      unexpected response shape.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```


---

## 4.4 4xx — You Messed Up

**These are the codes you'll use MOST as an API developer.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  CLIENT ERROR CODES (4xx)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  400 Bad Request                                                │
│  ───────────────                                                │
│  "I can't understand your request. The format is wrong."        │
│  Examples: Malformed JSON, missing required fields.             │
│  Library: "I can't read this form. Please fill it out again."   │
│                                                                 │
│  401 Unauthorized                                               │
│  ────────────────                                               │
│  "Who are you? I need to see identification."                   │
│  No credentials provided, or credentials are invalid.           │
│  Library: "You haven't shown your library card."                │
│  ⚠️ Misleading name — it really means UNAUTHENTICATED.          │
│                                                                 │
│  403 Forbidden                                                  │
│  ─────────────                                                  │
│  "I know who you are, but you're not ALLOWED to do this."       │
│  You authenticated, but lack permission.                        │
│  Library: "Your basic card doesn't grant access to              │
│           the rare manuscripts section."                        │
│                                                                 │
│  404 Not Found                                                  │
│  ─────────────                                                  │
│  "That resource doesn't exist."                                 │
│  Library: "We have no book with catalog number #99999."         │
│                                                                 │
│  405 Method Not Allowed                                         │
│  ──────────────────────                                         │
│  "This endpoint exists, but not for that method."               │
│  Library: "You can READ books here, not DONATE them.            │
│           Donations are at the other desk."                     │
│  Example: You sent DELETE to an endpoint that only accepts GET. │
│                                                                 │
│  409 Conflict                                                   │
│  ────────────                                                   │
│  "Your request conflicts with the current state."               │
│  Library: "You're trying to register a book with an ISBN        │
│           that already exists in our catalog."                  │
│                                                                 │
│  422 Unprocessable Entity                                       │
│  ────────────────────────                                       │
│  "I understand the format, but the content doesn't make sense." │
│  Example: JSON is valid, but age is -5 or email has no @.       │
│  Library: "Your form is legible, but the date you wrote         │
│           is February 30th. That doesn't exist."                │
│  ⚠️ FastAPI uses 422 heavily for Pydantic validation errors.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The critical distinction — 401 vs 403:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     401 vs 403                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  401: "WHO are you?"                                            │
│        │                                                        │
│        ├─ No library card shown at all                          │
│        ├─ Card is expired                                       │
│        └─ Card is fake                                          │
│                                                                 │
│        Fix: Log in. Provide valid credentials.                  │
│                                                                 │
│                                                                 │
│  403: "I know who you are, but NO."                             │
│        │                                                        │
│        ├─ Valid card, but basic tier → trying to enter VIP      │
│        ├─ Regular user → trying to access admin panel           │
│        └─ Read-only key → trying to delete data                 │
│                                                                 │
│        Fix: Get higher permissions. Or don't try this.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**400 vs 422 — another common confusion:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     400 vs 422                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  400: "I can't even PARSE what you sent."                       │
│        │                                                        │
│        ├─ Invalid JSON:  {name: Alice}  (missing quotes)        │
│        ├─ Wrong content type: sent XML, expected JSON           │
│        └─ Completely garbled data                               │
│                                                                 │
│                                                                 │
│  422: "I can parse it, but the VALUES are wrong."               │
│        │                                                        │
│        ├─ Valid JSON, but age is -5                             │
│        ├─ Valid JSON, but required field "email" is missing     │
│        └─ Valid JSON, but "status" must be one of [a, b, c]     │
│                                                                 │
│  Think of it this way:                                          │
│  400 = syntax error (can't read the form)                       │
│  422 = semantic error (can read it, but it's nonsense)          │
│                                                                 │
│  FastAPI + Pydantic (Week 2 Lecture 3) will handle              │
│  validation and return 422 automatically.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  415 Unsupported Media Type                                     │
│  ───────────────────────────                                    │
│  "The FORMAT you sent is one I cannot handle."                  │
│  Example: You sent Content-Type: text/xml to an endpoint        │
│           that only accepts application/json.                   │
│  Library: "We don't accept fax transmissions. Printed forms     │
│            only."                                               │
│                                                                 │
│  Distinction from 400:                                          │
│  ├─ 400 — the body is unparseable (malformed JSON)              │
│  ├─ 415 — the CONTENT TYPE is wrong (the body may be perfectly  │
│  │        valid XML — this endpoint simply does not speak XML)  │
│  └─ 422 — the type is correct, the VALUES are invalid           │
│                                                                 │
│                                                                 │
│  429 Too Many Requests                                          │
│  ──────────────────────                                         │
│  "You are sending too many requests. Slow down."                │
│  Rate limiting — the server is throttling a client that has     │
│  exceeded its allowed request quota in a given time window.     │
│  Library: "You have requested 100 books today. Your daily       │
│            limit is 100. Come back tomorrow."                   │
│                                                                 │
│  Standard response headers:                                     │
│  Retry-After: 60             ← try again in 60 seconds          │
│  X-RateLimit-Limit: 100      ← max requests per window          │
│  X-RateLimit-Remaining: 0    ← you have used them all           │
│  X-RateLimit-Reset: 1739094600 ← Unix timestamp: window resets  │
│                                                                 │
│  ⚠️ You will implement rate limiting with slowapi in            │
│     Week 12, Lecture 4.                                         │
│     When CONSUMING external APIs (Week 8), always inspect       │
│     these headers before retrying — hammering a rate-limited    │
│     endpoint extends your ban window.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Handling 429 correctly when consuming an external API:**

```python
import asyncio
import httpx

async def fetch_with_backoff(url: str, client: httpx.AsyncClient) -> dict:
    """Respect the Retry-After header instead of hammering the server."""
    response = await client.get(url)

    if response.status_code == 429:
        retry_after = int(response.headers.get("Retry-After", 60))
        print(f"Rate limited. Waiting {retry_after}s...")
        await asyncio.sleep(retry_after)  # Non-blocking wait
        response = await client.get(url)  # One retry

    response.raise_for_status()
    return response.json()
```
---

## 4.5 5xx — We Messed Up

```
┌─────────────────────────────────────────────────────────────────┐
│                  SERVER ERROR CODES (5xx)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  500 Internal Server Error                                      │
│  ─────────────────────────                                      │
│  "Something went wrong on OUR side. Not your fault."            │
│  The catch-all server error. Unhandled exception, bug, crash.   │
│  Library: "Our computer just showed a blue screen."             │
│  ⚠️ If YOUR FastAPI app raises an unhandled exception,          │
│     the client sees 500. This is a BUG in your code.            │
│                                                                 │
│  502 Bad Gateway                                                │
│  ───────────────                                                │
│  "I'm a middleman, and the server behind me is broken."         │
│  Library: "I called the main branch for you, but their          │
│           phone line is dead."                                  │
│  Common when a reverse proxy (Nginx) can't reach your app.      │
│                                                                 │
│  503 Service Unavailable                                        │
│  ────────────────────────                                       │
│  "I'm overloaded or under maintenance. Try again later."        │
│  Library: "We're closed for renovations. Come back tomorrow."   │
│  The server is alive but can't handle the request right now.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The rule of thumb:**

> "4xx means the CLIENT did something wrong — fix the request. 5xx means the SERVER did something wrong — the client can retry or wait. This distinction is critical. If your API returns 500 for bad input instead of 400 or 422, that's a bug in YOUR error handling."

---

## 4.6 Status Codes as Exceptions + Error Response Structure (Connection to Week 1)

**Connection to what you've learned:**

> "Remember custom exception hierarchies from Week 1, Lecture 2? Status codes follow the EXACT same pattern. They form families with shared behavior."

```python
# Week 1, Lecture 2: You learned to build exception hierarchies
# Status codes map DIRECTLY to this pattern.

class HTTPError(Exception):
    """Base — something went wrong with the HTTP exchange"""
    def __init__(self, status_code: int, detail: str):
        self.status_code = status_code
        self.detail = detail

class ClientError(HTTPError):
    """4xx — the client made a mistake"""
    pass

class BadRequestError(ClientError):
    """400 — malformed request"""
    pass

class NotFoundError(ClientError):
    """404 — resource doesn't exist"""
    pass

class ForbiddenError(ClientError):
    """403 — not allowed"""
    pass

class ServerError(HTTPError):
    """5xx — the server failed"""
    pass

class InternalError(ServerError):
    """500 — unhandled server error"""
    pass
```

```
┌─────────────────────────────────────────────────────────────────┐
│          STATUS CODES = EXCEPTION HIERARCHY                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    HTTPError                                    │
│                   /         \                                   │
│           ClientError    ServerError                            │
│           (4xx)          (5xx)                                  │
│          /    |    \       |     \                              │
│       400   404   403    500    503                             │
│                                                                 │
│  Same pattern as your custom exceptions from Week 1.            │
│  Catch broadly (ClientError) or narrowly (NotFoundError).       │
│                                                                 │
│  try:                                                           │
│      response = make_request()                                  │
│  except NotFoundError:                                          │
│      # Handle 404 specifically                                  │
│  except ClientError:                                            │
│      # Handle any other 4xx                                     │
│  except ServerError:                                            │
│      # Handle 5xx — maybe retry                                 │
│                                                                 │
│  This is EXACTLY how you'll handle errors in your               │
│  FastAPI applications (Week 2, Lecture 4).                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
**Standardising Your Error Response Body**

> "The exception hierarchy tells you how to structure your code. But what does the actual JSON body look like to the client when something goes wrong? This matters as much as the status code. If every endpoint returns errors in a different shape, clients cannot write reliable error-handling code."

**The problem — inconsistent error shapes:**

```python
# Three different endpoints, three incompatible error shapes.
# The API consumer must write different parsing logic for each.

# Endpoint A:
{"detail": "User not found"}

# Endpoint B:
{"message": "Invalid email", "code": 400}

# Endpoint C:
{"error": "Forbidden", "path": "/admin"}
```

**A consistent error response standard:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 ERROR RESPONSE STRUCTURE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP/1.1 422 Unprocessable Entity                              │
│  Content-Type: application/json                                 │
│                                                                 │
│  {                                                              │
│    "error": {                                                   │
│      "code": "VALIDATION_ERROR",   ← machine-readable string    │
│      "message": "Request validation failed", ← human-readable   │
│      "details": [                  ← field-level breakdown      │
│        {                                                        │
│          "field": "email",                                      │
│          "message": "Invalid email format"                      │
│        },                                                       │
│        {                                                        │
│          "field": "age",                                        │
│          "message": "Must be at least 18"                       │
│        }                                                        │
│      ]                                                          │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  WHY THIS STRUCTURE?                                            │
│  ├─ "code"    — clients switch on this string in code, not      │
│  │              the human message (messages change; codes don't) │
│  ├─ "message" — humans read this in logs and dashboards         │
│  ├─ "details" — pinpoints exactly what failed at field level    │
│  └─ Consistent envelope — same shape for every error type       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**More examples using the same envelope:**

```python
# 404 Not Found
{
    "error": {
        "code": "RESOURCE_NOT_FOUND",
        "message": "Book with id=42 does not exist",
        "details": []
    }
}

# 409 Conflict
{
    "error": {
        "code": "DUPLICATE_EMAIL",
        "message": "An account with this email already exists",
        "details": [
            {"field": "email", "message": "alice@example.com is already registered"}
        ]
    }
}

# 500 Internal Server Error — NEVER expose internals
{
    "error": {
        "code": "INTERNAL_ERROR",
        "message": "An unexpected error occurred. Please try again.",
        "details": []
    }
}
# ⚠️ Stack traces, database error messages, and internal paths
# must NEVER appear in 5xx responses. Log them server-side.
# The client receives only a generic message.
```

```
┌─────────────────────────────────────────────────────────────────┐
│            FASTAPI DEFAULT vs YOUR STANDARD                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FastAPI + Pydantic already produce structured 422 errors:      │
│                                                                 │
│  {                                                              │
│    "detail": [                                                  │
│      {                                                          │
│        "loc": ["body", "email"],                                │
│        "msg": "value is not a valid email address",             │
│        "type": "value_error.email"                              │
│      }                                                          │
│    ]                                                            │
│  }                                                              │
│                                                                 │
│  FastAPI uses "detail" as the key, not an "error" wrapper.      │
│  In Week 2, Lecture 4 you will learn to override this with      │
│  @app.exception_handler to enforce your own standard.           │
│  For now: understand WHY a consistent structure matters.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏️  PRACTICE CHECKPOINT 4.6                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q1. A POST /users request fails because the email already      │
│      exists. Design the HTTP status code and the full error     │
│      response body using the standard above.                    │
│                                                                 │
│  Q2. An unhandled exception fires in a database query. What     │
│      status code and body should the client receive?            │
│      What must you never include?                               │
│                                                                 │
│  Q3. Why use a string "code" field like "DUPLICATE_EMAIL"       │
│      instead of simply repeating the integer status code?       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
```
┌─────────────────────────────────────────────────────────────────┐
│                   ✅  CHECKPOINT SOLUTIONS 4.6                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A1. 409 Conflict.                                              │
│      {"error": {"code": "DUPLICATE_EMAIL",                      │
│                 "message": "This email is already registered",  │
│                 "details": [{"field": "email",                  │
│                              "message": "Already in use"}]}}    │
│                                                                 │
│  A2. 500 Internal Server Error. Body: a generic message only.   │
│      Never expose the stack trace, exception text, SQL query,   │
│      table name, or any internal path. These reveal your        │
│      architecture and may expose vulnerabilities.               │
│      {"error": {"code": "INTERNAL_ERROR",                       │
│                 "message": "An unexpected error occurred",       │
│                 "details": []}}                                 │
│                                                                 │
│  A3. Multiple distinct errors share the same status code.       │
│      Both "invalid email" and "password too short" are 422.     │
│      A machine-readable code lets clients branch logic exactly  │
│      without parsing human strings that may be translated,      │
│      reworded, or change between releases.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

# PART 5: HEADERS — THE METADATA LAYER

## 5.1 What Are Headers?

**Headers are key-value metadata attached to requests and responses.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     WHAT ARE HEADERS?                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Headers are METADATA — information ABOUT the message,          │
│  not the message itself.                                        │
│                                                                 │
│  ┌───────────────────────────────────────────────────────┐      │
│  │  POST /books HTTP/1.1          ← What you want        │      │
│  │  Host: library.example.com     ← Header (where)       │      │
│  │  Content-Type: application/json← Header (format)      │      │
│  │  Authorization: Bearer abc123  ← Header (who)         │      │
│  │  Accept-Language: en           ← Header (preference)  │      │
│  │                                                       │      │
│  │  {"title": "Clean Code"}       ← Body (the content)   │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                 │
│  Library analogy:                                               │
│  The BODY is the actual book you're donating.                   │
│  The HEADERS are everything written on the donation form:       │
│  ├─ Your library card number (Authorization)                    │
│  ├─ "This book is in English" (Content-Type / Language)         │
│  ├─ "I'd like the confirmation in French" (Accept-Language)     │
│  └─ "This is for the Main Street branch" (Host)                 │
│                                                                 │
│  Format: Key: Value (separated by colon, one per line)          │
│  Headers are CASE-INSENSITIVE: Content-Type = content-type      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Content-Type (The Language Agreement)

**The most important header. It tells the other side WHAT FORMAT the body is in.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONTENT-TYPE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "The body of this message is formatted as ____."               │
│                                                                 │
│  Common values:                                                 │
│  ├─ application/json         ← JSON (the one you'll use most)   │
│  ├─ text/html                ← HTML web pages                   │
│  ├─ text/plain               ← Plain text                       │
│  ├─ multipart/form-data      ← File uploads                     │
│  └─ application/octet-stream ← Raw binary data                  │
│                                                                 │
│  For API development, it's almost always application/json.      │
│                                                                 │
│                                                                 │
│  Library analogy:                                               │
│  "This book is written in English."                             │
│  "This book is in Braille."                                     │
│  "This is a DVD, not a book."                                   │
│                                                                 │
│  If the librarian expects a book and you hand them a DVD,       │
│  they can't process it. Content-Type prevents that mismatch.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The matching pair: Accept**

```
Content-Type:  "The body I'm SENDING is in this format."
Accept:        "The body I WANT BACK should be in this format."
```

```
# Request: "I'm sending JSON, and I want JSON back"
POST /books HTTP/1.1
Content-Type: application/json     ← my body is JSON
Accept: application/json           ← I want JSON back

{"title": "Clean Code"}
```

---

## 5.3 Authorization (Proving Who You Are)

```
┌─────────────────────────────────────────────────────────────────┐
│                    AUTHORIZATION                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "Here are my credentials."                                     │
│                                                                 │
│  Common patterns:                                               │
│                                                                 │
│  Bearer Token (most common in modern APIs):                     │
│  Authorization: Bearer eyJhbGciOiJIUzI1NiIs...                  │
│                                                                 │
│  Basic Auth (username:password, base64 encoded):                │
│  Authorization: Basic dXNlcjpwYXNzd29yZA==                      │
│                                                                 │
│  API Key (sometimes in Authorization, sometimes custom header): │
│  Authorization: ApiKey sk-abc123...                             │
│  X-API-Key: sk-abc123...                                        │
│                                                                 │
│                                                                 │
│  Library analogy:                                               │
│  Bearer Token = Your digital library card (contains your ID,    │
│                 membership level, expiration date — all encoded)│
│  Basic Auth   = Writing your name and password on the form      │
│  API Key      = A special access code for automated systems     │
│                                                                 │
│  ⚠️ We'll implement auth properly in Week 5/6 (JWT).            │
│     For now, just understand the HEADER FORMAT.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Cookies — The Browser's Built-In State Mechanism**

> "The Authorization header covers how APIs authenticate. But browsers have a second, older mechanism that is equally important: cookies. Understanding both is essential — cookie-based auth is what protects the web apps you use every day."

```
┌─────────────────────────────────────────────────────────────────┐
│                    HOW COOKIES WORK                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A cookie is a small piece of data the SERVER sends to the      │
│  browser. The browser stores it and AUTOMATICALLY attaches it   │
│  to every subsequent request to the same domain — no code       │
│  required on the client side.                                   │
│                                                                 │
│  SERVER → browser (response header):                            │
│  Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax  │
│                                                                 │
│  BROWSER → server (on every matching subsequent request):       │
│  Cookie: session_id=abc123                                      │
│                                                                 │
│  Key difference from Authorization header:                      │
│  Authorization: Bearer ...  ← your JavaScript code adds this   │
│  Cookie: ...                ← browser adds this AUTOMATICALLY   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The Set-Cookie security attributes:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SET-COOKIE ATTRIBUTES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax;    │
│              Max-Age=3600; Path=/; Domain=example.com           │
│                                                                 │
│  HttpOnly                                                       │
│  ├─ JavaScript CANNOT read this cookie (document.cookie fails)  │
│  └─ Critical protection: prevents XSS attacks from stealing     │
│     session tokens via injected scripts                         │
│                                                                 │
│  Secure                                                         │
│  ├─ Browser sends this cookie ONLY over HTTPS                   │
│  └─ Never transmitted over plain HTTP                           │
│     ⚠️ http://localhost will NOT send Secure cookies — this is  │
│     expected. Remove Secure for local dev if needed.            │
│                                                                 │
│  SameSite=Lax   (recommended default)                           │
│  ├─ Sent on same-site requests and top-level navigations        │
│  ├─ NOT sent on cross-site sub-requests (primary CSRF defence)  │
│  └─ Options: Strict (never cross-site), Lax, None (cross-site)  │
│     SameSite=None requires Secure to be set simultaneously      │
│                                                                 │
│  Max-Age=3600                                                   │
│  └─ Cookie expires after 3600 seconds (1 hour)                  │
│     Browser automatically deletes it after expiry              │
│                                                                 │
│  Path=/                                                         │
│  └─ Send this cookie for all URL paths on the domain            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Cookie-based sessions vs Bearer token auth — a direct comparison:**

```
┌─────────────────────────────────────────────────────────────────┐
│           COOKIE SESSION vs BEARER TOKEN                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  COOKIE-BASED SESSION                                           │
│  1. Login → server creates a session record in DB or Redis      │
│  2. Server sends: Set-Cookie: session_id=<opaque_id>; HttpOnly  │
│  3. Browser attaches Cookie: session_id=... automatically       │
│  4. Server looks up session_id to get the user's identity       │
│                                                                 │
│  ✅ Automatic — browser handles storage and sending             │
│  ✅ HttpOnly prevents JS from reading the token                 │
│  ❌ Does not work naturally for mobile apps                     │
│  ❌ Requires server-side session storage (Redis/DB)             │
│                                                                 │
│                                                                 │
│  BEARER TOKEN (JWT)                                             │
│  1. Login → server creates and returns a signed JWT             │
│  2. Client stores JWT (localStorage, mobile secure keychain)   │
│  3. Client explicitly adds: Authorization: Bearer <jwt>         │
│  4. Server validates the JWT's signature — no storage lookup    │
│                                                                 │
│  ✅ Stateless — no server-side session storage needed           │
│  ✅ Works naturally for mobile apps and cross-domain calls      │
│  ❌ JS must read the token to send it (XSS risk if in storage)  │
│                                                                 │
│                                                                 │
│  This course uses JWT (Bearer tokens). Full implementation in   │
│  Week 9. The HttpOnly-cookie-stored JWT hybrid (the best of     │
│  both) is also covered in Week 9, Lecture 3.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Seeing cookies live with curl:**

```bash
# Step 1: Server sets a cookie, save it to a cookie jar file
curl -v -c cookies.txt \
  "https://httpbin.org/cookies/set/session_id/abc123"

# Look for in the response output:
# < Set-Cookie: session_id=abc123; Path=/

# Step 2: Resend with the cookie jar — browser-like behaviour
curl -v -b cookies.txt \
  "https://httpbin.org/cookies"

# Look for in the request output:
# > Cookie: session_id=abc123   ← sent automatically from the jar
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏️  PRACTICE CHECKPOINT 5.3                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q1. A session cookie has HttpOnly set. Can a JavaScript        │
│      XSS attack steal it via document.cookie? Why or why not?  │
│                                                                 │
│  Q2. You are building a mobile banking app. Should it use       │
│      cookie sessions or JWT Bearer tokens? Give two reasons.   │
│                                                                 │
│  Q3. A developer sets SameSite=None so the API works from any  │
│      frontend domain. What security risk does this introduce,   │
│      and what other attribute is required alongside it?         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
```
┌─────────────────────────────────────────────────────────────────┐
│                   ✅  CHECKPOINT SOLUTIONS 5.3                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A1. No. HttpOnly blocks JavaScript's access to the cookie.     │
│      document.cookie does not expose HttpOnly cookies.          │
│      The browser sends it to the server, but no script —        │
│      including injected malicious ones — can read it.           │
│                                                                 │
│  A2. JWT Bearer tokens. Reasons:                                │
│      1. Mobile apps have no built-in cookie jar the way         │
│         browsers do. JWT stored in a device secure keychain     │
│         is the platform-native pattern.                         │
│      2. Mobile apps often call APIs across multiple domains;    │
│         cookies are domain-scoped and awkward across services.  │
│                                                                 │
│  A3. SameSite=None disables CSRF protection — a malicious page  │
│      can trick a logged-in browser into sending the cookie to   │
│      your API (cross-site request forgery). Required alongside: │
│      Secure must also be present. Browsers reject              │
│      SameSite=None without Secure entirely.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

## 5.4 Other Essential Headers

```
┌─────────────────────────────────────────────────────────────────┐
│                  OTHER COMMON HEADERS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REQUEST HEADERS (client → server):                             │
│  ──────────────────────────────────                             │
│  Host: api.example.com                                          │
│  └─ REQUIRED in HTTP/1.1. Which server you're talking to.       │
│     (One IP can host multiple domains.)                         │
│                                                                 │
│  User-Agent: Mozilla/5.0 ... (or) python-httpx/0.25             │
│  └─ Identifies the client software. Servers use this for        │
│     analytics, compatibility, or bot detection.                 │
│                                                                 │
│  Accept: application/json                                       │
│  └─ What response format the client prefers.                    │
│                                                                 │
│                                                                 │
│  RESPONSE HEADERS (server → client):                            │
│  ──────────────────────────────────                             │
│  Content-Length: 256                                            │
│  └─ Size of the response body in bytes.                         │
│                                                                 │
│  Date: Mon, 09 Feb 2026 10:30:00 GMT                            │
│  └─ When the server generated the response.                     │
│                                                                 │
│  Cache-Control: max-age=3600                                    │
│  └─ How long the client can cache this response.                │
│                                                                 │
│  Location: /books/99                                            │
│  └─ Used with 201 Created — where the new resource lives.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

**Caching Headers — Saving Bandwidth and Reducing Latency**

> "Every time your API returns the same data that hasn't changed, the client is paying the same network round-trip cost. Caching headers let you tell clients — and intermediary proxies and CDNs — how long a response stays fresh, and how to check whether it is still valid."

**Cache-Control directives — the full picture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    CACHE-CONTROL DIRECTIVES                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Cache-Control: max-age=3600                                    │
│  └─ "Fresh for 3600 seconds. Client may reuse without asking."  │
│                                                                 │
│  Cache-Control: no-cache                                        │
│  └─ "Store it, but always revalidate before using."             │
│     Client must send a conditional request (ETag/If-None-Match) │
│     to confirm freshness before using the cached copy.          │
│     ⚠️ Confusing name: it does NOT mean "never cache".          │
│                                                                 │
│  Cache-Control: no-store                                        │
│  └─ "Do not store this response anywhere, ever."                │
│     Use for sensitive data: bank balances, session state.       │
│     This IS the "never cache" directive.                        │
│                                                                 │
│  Cache-Control: private                                         │
│  └─ "Only the end user's browser may cache this."               │
│     CDNs and shared proxies must not store it.                  │
│     Use for user-specific data.                                 │
│                                                                 │
│  Cache-Control: public                                          │
│  └─ "Any cache — browser, CDN, proxy — may store this."         │
│     Use for shared, public, user-agnostic responses.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**ETag and Conditional Requests — smart cache validation:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ETAG MECHANISM                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ETag = "Entity Tag" — a hash/fingerprint of the response body  │
│                                                                 │
│  FIRST REQUEST:                                                 │
│  GET /books/42 HTTP/1.1                                         │
│                                                                 │
│  HTTP/1.1 200 OK                                                │
│  ETag: "33a64df5"          ← server sends a content fingerprint │
│  Cache-Control: max-age=60                                      │
│  {"id": 42, "title": "Clean Code"}                              │
│                                                                 │
│                                                                 │
│  60 SECONDS LATER (cache expired, client revalidates):          │
│  GET /books/42 HTTP/1.1                                         │
│  If-None-Match: "33a64df5"  ← "Is my version still current?"   │
│                                                                 │
│  UNCHANGED → 304 Not Modified (empty body — client uses cache)  │
│  HTTP/1.1 304 Not Modified                                      │
│  ETag: "33a64df5"                                               │
│  ← No body. Saves full bandwidth.                               │
│                                                                 │
│  CHANGED → 200 OK (full new response)                           │
│  HTTP/1.1 200 OK                                                │
│  ETag: "a1b2c3d4"          ← new fingerprint                    │
│  {"id": 42, "title": "Clean Code (2nd ed)"}                     │
│                                                                 │
│                                                                 │
│  Last-Modified / If-Modified-Since (timestamp equivalent):      │
│  ├─ Server sends: Last-Modified: Tue, 18 Feb 2026 12:00:00 GMT  │
│  ├─ Client sends: If-Modified-Since: Tue, 18 Feb 2026 ...       │
│  └─ ETag is more precise; Last-Modified is simpler to implement │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Choosing Cache-Control in practice:**

```python
from fastapi.responses import JSONResponse

# Public, rarely-changing resource — CDN can cache globally
response = JSONResponse(
    content={"books": [...]},
    headers={"Cache-Control": "public, max-age=86400"}  # 1 day
)

# User-specific data — browser only, short window
response = JSONResponse(
    content={"balance": 1024.50},
    headers={"Cache-Control": "private, no-store"}
)

# Real-time endpoint — never cache
response = JSONResponse(
    content={"online_users": 4821},
    headers={"Cache-Control": "no-store"}
)
```

---

**Compression Headers — Smaller Responses, Faster Transfers**

```
┌─────────────────────────────────────────────────────────────────┐
│                   COMPRESSION HEADERS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client declares what it can decompress (request):              │
│  Accept-Encoding: gzip, deflate, br                             │
│  └─ "I support gzip, deflate, and Brotli compression"           │
│     br = Brotli — newer, better compression ratio than gzip     │
│                                                                 │
│  Server compresses and declares format used (response):         │
│  Content-Encoding: gzip                                         │
│  └─ "The body I am sending is gzip-compressed."                 │
│     Client decompresses automatically before reading it.        │
│                                                                 │
│                                                                 │
│  REAL IMPACT:                                                   │
│  ├─ A 100 KB JSON response → ~15 KB with gzip                   │
│  └─ ~85% bandwidth reduction on repetitive text-based data      │
│                                                                 │
│                                                                 │
│  WHEN NOT TO COMPRESS:                                          │
│  └─ Already-compressed formats: JPEG, PNG, MP4, ZIP             │
│     Compressing them again produces LARGER output.              │
│     Content-Encoding only applies to text: JSON, HTML, CSS, JS  │
│                                                                 │
│                                                                 │
│  IN PRODUCTION:                                                 │
│  ├─ Nginx or your cloud load balancer handles this for you      │
│  └─ FastAPI provides GzipMiddleware for app-level compression   │
│     (Week 12, Lecture 3)                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Ask the server for gzip-compressed response
curl -H "Accept-Encoding: gzip" -v https://httpbin.org/get

# Look for in response:
# < Content-Encoding: gzip

# --compressed tells curl to decompress before printing output
curl --compressed https://httpbin.org/get
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏️  PRACTICE CHECKPOINT 5.4                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q1. Your GET /countries endpoint returns a list of all         │
│      countries — data that changes at most once a year.         │
│      Write the ideal Cache-Control header value and justify     │
│      each directive you chose.                                  │
│                                                                 │
│  Q2. A client sends If-None-Match: "abc123" for a resource      │
│      that HAS changed since that ETag was generated.            │
│      What status code and body does the server return?          │
│                                                                 │
│  Q3. What is the functional difference between                  │
│      Cache-Control: no-cache and Cache-Control: no-store?       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
```
┌─────────────────────────────────────────────────────────────────┐
│                   ✅  CHECKPOINT SOLUTIONS 5.4                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A1. Cache-Control: public, max-age=31536000                    │
│      public → CDNs may cache it (not user-specific)             │
│      max-age=31536000 → fresh for ~1 year (data rarely changes) │
│      Optional addition: pair with ETag so you can force-expire  │
│      the cache early if a country changes its name.             │
│                                                                 │
│  A2. 200 OK with the full new response body and a new ETag.     │
│      HTTP/1.1 200 OK                                            │
│      ETag: "def456"                                             │
│      Content-Type: application/json                             │
│      {... updated data ...}                                     │
│                                                                 │
│  A3. no-cache: the response IS stored, but the client must      │
│      revalidate with the server (using If-None-Match) before    │
│      using it, even if max-age hasn't expired yet.              │
│      no-store: the response must NOT be stored at all — not in  │
│      the browser cache, not in any intermediate proxy or CDN.   │
│      no-cache = "store but always check". no-store = "never     │
│      touch a cache at all".                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 Custom Headers

```
┌─────────────────────────────────────────────────────────────────┐
│                    CUSTOM HEADERS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You CAN create your own headers for your API.                  │
│  Convention: prefix with X- (historical) or just use a clear    │
│  name with your org prefix.                                     │
│                                                                 │
│  Examples:                                                      │
│  ├─ X-Request-ID: 550e8400-e29b-41d4-a716-446655440000          │
│  │  └─ Unique ID for tracing a request through systems          │
│  │                                                              │
│  ├─ X-Rate-Limit-Remaining: 95                                  │
│  │  └─ How many requests you have left before throttling        │
│  │                                                              │
│  └─ X-Correlation-ID: abc-123-def                               │
│     └─ Links related requests across microservices              │
│                                                                 │
│  ⚠️ Don't put BUSINESS DATA in headers. That goes in the body.  │
│  Headers are for METADATA: identifiers, formats, auth,          │
│  caching hints, tracing information.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Modeling headers in Python (connection to Week 1 type hints):**

```python
# Headers are just key-value pairs — dicts with string keys and values
# This should feel familiar after Week 1, Lecture 1

from typing import Optional

# Type alias for clarity (Week 1, Lecture 1)
Headers = dict[str, str]

def build_request_headers(
    content_type: str = "application/json",
    token: Optional[str] = None,
    request_id: Optional[str] = None,
) -> Headers:
    """Build headers for an HTTP request"""
    headers: Headers = {
        "Content-Type": content_type,
        "Accept": "application/json",
    }
    if token is not None:
        headers["Authorization"] = f"Bearer {token}"
    if request_id is not None:
        headers["X-Request-ID"] = request_id
    return headers

# Usage
headers = build_request_headers(token="abc123", request_id="req-001")
# {'Content-Type': 'application/json', 'Accept': 'application/json',
#  'Authorization': 'Bearer abc123', 'X-Request-ID': 'req-001'}
```
---
## 5.6 HTTPS and TLS — The Security Layer

**The problem with everything covered so far:**

> "Every HTTP message in this lecture — every header, every body, every token in the Authorization header — has been plain, human-readable text. If someone sits between the client and server on the same network — a café Wi-Fi, a compromised router, an ISP — they can read all of it."

```
┌─────────────────────────────────────────────────────────────────┐
│               THE PLAIN HTTP PROBLEM                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Without encryption (http://):                                  │
│                                                                 │
│   CLIENT ────────────────────────────────── SERVER              │
│            ↑ an attacker watching this sees:                    │
│                                                                 │
│   POST /login HTTP/1.1                                          │
│   Content-Type: application/json                                │
│                                                                 │
│   {"email": "alice@example.com", "password": "hunter2"}  😱    │
│                                                                 │
│   GET /dashboard HTTP/1.1                                       │
│   Authorization: Bearer eyJhbGci...    ← token stolen           │
│                                                                 │
│   This is a man-in-the-middle (MITM) attack.                    │
│   Any device on the same network can intercept this             │
│   with basic, widely-available tools.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The solution: HTTPS = HTTP + TLS**

```
┌─────────────────────────────────────────────────────────────────┐
│                   WHAT HTTPS ADDS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP  = The text protocol you have been learning               │
│  HTTPS = HTTP wrapped inside a TLS encrypted tunnel             │
│                                                                 │
│  TLS (Transport Layer Security) provides THREE guarantees:      │
│                                                                 │
│  1. ENCRYPTION                                                  │
│     Data is scrambled in transit using a shared secret key.     │
│     The attacker sees: [meaningless bytes]                      │
│     Not: {"password": "hunter2"}                                │
│                                                                 │
│  2. AUTHENTICATION (server identity)                            │
│     You verify you are talking to the REAL server —             │
│     not an impersonator that intercepted the connection.        │
│     Proof: the server presents a signed TLS Certificate.        │
│                                                                 │
│  3. INTEGRITY                                                   │
│     Data cannot be silently altered in transit.                 │
│     If one byte is modified, cryptographic verification fails   │
│     and the connection is terminated.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What is a TLS Certificate?**

```
┌─────────────────────────────────────────────────────────────────┐
│                    TLS CERTIFICATES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A certificate is a signed document that proves:                │
│  "This server genuinely owns the domain api.example.com."       │
│                                                                 │
│  A certificate contains:                                        │
│  ├─ The domain name (api.example.com)                           │
│  ├─ A public encryption key (used to set up the shared secret)  │
│  ├─ An expiry date                                              │
│  └─ A cryptographic SIGNATURE from a Certificate Authority (CA) │
│                                                                 │
│  Certificate Authority (CA):                                    │
│  └─ A trusted third party (Let's Encrypt, DigiCert, Comodo)    │
│     that verified the domain owner's identity and countersigned │
│     the certificate. Your OS and browser pre-trust a list of   │
│     approved CAs. If the CA signed it, the browser trusts it.   │
│                                                                 │
│  Library analogy:                                               │
│  Before handing your library card to the person at the desk,   │
│  you check their government-issued ID badge.                    │
│  The government (CA) vouches for the librarian (server).        │
│  An expired or fake badge → you walk away.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The TLS Handshake — simplified to what matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TLS HANDSHAKE (CONCEPTUAL)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT                                         SERVER          │
│                                                                 │
│  1. "Hello. I support TLS 1.3.                                  │
│      Here are cipher suites I know."                            │
│  ─────────────────────────────────────────────▶                 │
│                                                                 │
│  2. "Hello. Chosen cipher: AES-256-GCM.                         │
│      Here is my CERTIFICATE. Verify it."                        │
│  ◀─────────────────────────────────────────────                 │
│                                                                 │
│  3. Client verifies the certificate:                            │
│     ├─ Domain matches the one in the address bar? ✅            │
│     ├─ Not expired? ✅                                          │
│     └─ CA signature trusted by my OS? ✅                        │
│                                                                 │
│  4. Client and server derive a SHARED SECRET                    │
│     using asymmetric cryptography.                              │
│     Neither side ever transmits the secret itself.              │
│                                                                 │
│  5. All further communication — the HTTP request and response   │
│     — travels encrypted inside this tunnel.                     │
│     An attacker sees only ciphertext.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**http:// vs https:// in your daily work:**

```
┌─────────────────────────────────────────────────────────────────┐
│             HTTP vs HTTPS — PRACTICAL REALITY                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  http://localhost:8000        ← Local development only          │
│  └─ Traffic never leaves your machine (loopback interface).     │
│     Man-in-the-middle is physically impossible.                 │
│     Plain HTTP is fine. Secure cookies will not send, but      │
│     that is expected behaviour for local dev.                   │
│                                                                 │
│  https://api.yourapp.com      ← Production — always HTTPS       │
│  └─ Real network. Real interception risk.                       │
│     Serving auth endpoints over HTTP in production is a         │
│     critical, unacceptable security vulnerability.              │
│                                                                 │
│                                                                 │
│  PORT CONVENTIONS:                                              │
│  ├─ HTTP  default: port 80                                      │
│  └─ HTTPS default: port 443                                     │
│     https://example.com = https://example.com:443               │
│                                                                 │
│                                                                 │
│  WHO HANDLES TLS IN A REAL DEPLOYMENT:                          │
│                                                                 │
│  Internet ──HTTPS──▶ Nginx / Load Balancer ──HTTP──▶ FastAPI    │
│                          (TLS termination)                      │
│                                                                 │
│  FastAPI speaks plain HTTP internally.                          │
│  Nginx handles the certificates and decryption.                 │
│  Your FastAPI code is not aware of TLS at all.                  │
│  Week 15, Lecture 1 covers this deployment architecture.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Browser signals:**

```
🔒  Padlock icon in address bar    = HTTPS + valid certificate
⚠️  "Not Secure" warning          = plain HTTP connection
🔴  "Your connection is not private" = certificate error
                                     (expired / domain mismatch /
                                      untrusted CA)
```

> "From this point forward, whenever this course shows `https://`, you know exactly what is happening: your HTTP text message is wrapped inside a TLS encrypted tunnel, with the server's identity verified by a Certificate Authority before a single byte of data is exchanged."

> "This is also why the Bearer tokens you will build in Week 9 are safe in production: they travel inside that encrypted tunnel. Sending authentication tokens over plain HTTP in production is not a configuration oversight — it is a vulnerability."

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏️  PRACTICE CHECKPOINT 5.6                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q1. Your FastAPI production server sends Bearer tokens in the  │
│      Authorization header over plain HTTP (http://).            │
│      Name the attack and describe what an attacker can do.      │
│                                                                 │
│  Q2. List TLS's three security guarantees. For each, state the  │
│      concrete consequence of NOT having it.                     │
│                                                                 │
│  Q3. A teammate says: "We need HTTPS even for localhost."        │
│      Are they right? Give a case where they would be right      │
│      and a case where plain HTTP is acceptable.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
```
┌─────────────────────────────────────────────────────────────────┐
│                   ✅  CHECKPOINT SOLUTIONS 5.6                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A1. Man-in-the-middle (MITM) attack. Any device on the same    │
│      network path — café Wi-Fi router, ISP, compromised switch  │
│      — can read the Authorization header, extract the token,    │
│      and use it to impersonate the user: read their data,       │
│      modify their account, perform actions under their identity. │
│                                                                 │
│  A2. Encryption → without it, passwords, tokens, and private   │
│      data are visible in plain text to any eavesdropper.        │
│      Authentication → without it, an attacker impersonates the  │
│      server, capturing credentials entered by the user (MITM    │
│      phishing attack).                                          │
│      Integrity → without it, an attacker can modify data        │
│      mid-transit — e.g. change a payment amount from 1 to 1000. │
│                                                                 │
│  A3. Usually wrong — localhost traffic never leaves the machine, │
│      so MITM is impossible and plain HTTP is fine.              │
│      Right when: testing Secure cookie behaviour (they refuse    │
│      to send over HTTP), or testing HSTS headers, or running    │
│      a feature that explicitly checks for HTTPS in the URL.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 6: REST — THE ARCHITECTURE

## 6.1 What REST Actually Means

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHAT IS REST?                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REST = Representational State Transfer                         │
│                                                                 │
│  REST is NOT a standard. NOT a library. NOT a protocol.         │
│  It is an ARCHITECTURAL STYLE — a set of principles for         │
│  designing networked applications.                              │
│                                                                 │
│  Defined by Roy Fielding in his 2000 PhD dissertation.          │
│  He was describing how the web ALREADY worked —                 │
│  then people started applying those principles to APIs.         │
│                                                                 │
│                                                                 │
│  A "RESTful API" is an API that follows these principles:       │
│                                                                 │
│  1. Everything is a RESOURCE (thing you can name)               │
│  2. Each resource has a unique URL (address)                    │
│  3. You interact using STANDARD METHODS (GET, POST, PUT...)     │
│  4. The server is STATELESS (no memory between requests)        │
│  5. Responses can be CACHED                                     │
│  6. The interface is UNIFORM (consistent, predictable)          │
│                                                                 │
│  Let's unpack these one by one.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 Everything is a Resource

```
┌─────────────────────────────────────────────────────────────────┐
│                  EVERYTHING IS A RESOURCE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A RESOURCE is any THING that can be named.                     │
│                                                                 │
│  In a library:                                                  │
│  ├─ A book is a resource                                        │
│  ├─ A member is a resource                                      │
│  ├─ A loan (borrowing record) is a resource                     │
│  ├─ The collection of all books is ALSO a resource              │
│  └─ A search result is ALSO a resource                          │
│                                                                 │
│  In an API:                                                     │
│  ├─ A user           → /users/42                                │
│  ├─ All users        → /users                                   │
│  ├─ A user's orders  → /users/42/orders                         │
│  ├─ A specific order → /users/42/orders/7                       │
│  └─ A product        → /products/99                             │
│                                                                 │
│  Resources are NOUNS, not VERBS.                                │
│  /users       ✅  (noun — the thing)                            │
│  /getUsers    ❌  (verb — an action)                            │
│  /createUser  ❌  (verb — the METHOD tells you the action)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.3 URLs as Resource Addresses

**A URL uniquely identifies a resource. Let's dissect one.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ANATOMY OF A URL                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  https://api.library.com:8080/books/42?format=full#chapter1     │
│  ──┬──   ───────┬──────── ─┬─ ───┬─── ─────┬───── ────┬────     │
│    │           │           │     │         │          │         │
│  scheme      host        port   path     query     fragment     │
│                                                                 │
│  scheme:    How to connect (http or https)                      │
│  host:      Which server (api.library.com)                      │
│  port:      Which door (8080; default is 80/443)                │
│  path:      Which resource (/books/42)                          │
│  query:     Extra parameters (?format=full)                     │
│  fragment:  Client-side only (#chapter1, never sent to server)  │
│                                                                 │
│                                                                 │
│  As an API developer, you mostly care about:                    │
│  ├─ PATH:  Identifies the resource                              │
│  └─ QUERY: Filters, pagination, options                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**URL patterns for a RESTful API:**

```
┌─────────────────────────────────────────────────────────────────┐
│                URL PATTERN CONVENTIONS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  /books              → The collection of all books              │
│  /books/42           → A specific book (ID = 42)                │
│  /books/42/reviews   → All reviews for book 42                  │
│  /books/42/reviews/7 → A specific review (ID = 7) for book 42   │
│                                                                 │
│  PATTERN:                                                       │
│  /collection                  → all items                       │
│  /collection/{id}             → one item                        │
│  /collection/{id}/sub         → related sub-collection          │
│  /collection/{id}/sub/{id}    → one sub-item                    │
│                                                                 │
│  FILTERING via query parameters:                                │
│  /books?author=martin         → books by Martin                 │
│  /books?available=true        → available books                 │
│  /books?page=2&limit=20       → pagination                      │
│  /books?sort=title&order=asc  → sorting                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
**URL Encoding — How Special Characters Travel Safely**

> "A URL can only legally contain a limited character set: letters, digits, and a handful of reserved characters like `/`, `?`, `&`, and `=`. All of these have structural meaning in a URL. When those characters appear *inside* data values — a name with an ampersand, an email address, a search query with spaces — they must be encoded. If they are not, the URL breaks."

```
┌─────────────────────────────────────────────────────────────────┐
│                    PERCENT-ENCODING                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A character is encoded as %XX where XX is its hex value.       │
│                                                                 │
│  Character  →  Encoded      Reason it needs encoding            │
│  ─────────     ────────     ────────────────────────────        │
│  Space      →  %20 (or +)   Spaces are not valid in URLs        │
│  &          →  %26          & separates query params            │
│  =          →  %3D          = pairs key/value in query strings  │
│  +          →  %2B          + means space in query strings      │
│  /          →  %2F          / separates URL path segments       │
│  ?          →  %3F          ? starts the query string           │
│  #          →  %23          # starts the fragment               │
│  @          →  %40          common in email addresses           │
│  %          →  %25          the escape character itself         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The encoding bug — what happens without it:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE ENCODING BUG                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Searching for author "Martin & Sons":                          │
│                                                                 │
│  ❌ UNENCODED (broken):                                         │
│  GET /books?author=Martin & Sons                                │
│                       ↑                                         │
│  Server sees TWO query params:                                  │
│      author = "Martin"                                          │
│      Sons   = (malformed second key with no value)              │
│                                                                 │
│  ✅ CORRECTLY ENCODED:                                          │
│  GET /books?author=Martin%20%26%20Sons                          │
│     or                                                          │
│  GET /books?author=Martin+%26+Sons                              │
│                                                                 │
│  Server correctly receives: author = "Martin & Sons"            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Python tools for URL encoding:**

```python
from urllib.parse import quote, quote_plus, urlencode, unquote

author = "Martin & Sons"

# Encode a path segment (spaces → %20)
print(quote(author))           # Martin%20%26%20Sons

# Encode a query string value (spaces → +, preferred for query strings)
print(quote_plus(author))      # Martin+%26+Sons

# Build a full query string from a dict — the correct way
params = {
    "author": "Martin & Sons",
    "available": True,
    "page": 2,
}
print(urlencode(params))
# author=Martin+%26+Sons&available=True&page=2

# Decode what a server receives
print(unquote("Martin%20%26%20Sons"))  # Martin & Sons

# Practical full URL construction
query_string = urlencode(params)
url = f"https://api.library.com/books?{query_string}"
```

**Libraries handle this automatically — but you must recognise it:**

```python
import httpx

# httpx URL-encodes params dict automatically
async with httpx.AsyncClient() as client:
    response = await client.get(
        "https://api.library.com/books",
        params={"author": "Martin & Sons", "available": True},
    )
    # httpx sends: /books?author=Martin+%26+Sons&available=True
    print(response.request.url)  # Shows the encoded URL
```

> "You will not manually call `quote()` often — httpx, requests, and browsers handle encoding automatically. But you MUST understand it to debug. When you see `%20` or `%26` in a server log, or a query parameter arrives as `Martin+%26+Sons`, you now know exactly what happened and why."

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏️  PRACTICE CHECKPOINT 6.3                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q1. A user searches for "C++ & Python" in your API.            │
│      Write the correctly percent-encoded query string.          │
│                                                                 │
│  Q2. You receive this URL in a server log:                      │
│      /books?title=Clean%20Code&tag=best%20practices             │
│      What are the decoded query parameter values?               │
│                                                                 │
│  Q3. A developer writes: f"/search?q={user_input}"              │
│      where user_input = "cats & dogs".                          │
│      What does the server receive for q, and how should         │
│      the developer fix it?                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
```
┌─────────────────────────────────────────────────────────────────┐
│                   ✅  CHECKPOINT SOLUTIONS 6.3                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A1. /books?query=C%2B%2B+%26+Python                            │
│      + signs → %2B (literal plus, not a space)                  │
│      &       → %26 (literal ampersand, not a separator)         │
│      space   → + (in query strings)                             │
│                                                                 │
│  A2. title = "Clean Code"                                       │
│      tag   = "best practices"                                   │
│      (%20 decodes to a space in both cases)                     │
│                                                                 │
│  A3. The URL becomes /search?q=cats & dogs — the & is parsed   │
│      as a query param separator. The server receives q="cats"   │
│      and a malformed second param "dogs". Fix: pass as a dict   │
│      to httpx params= or use urllib.parse.urlencode({"q":       │
│      user_input}) to produce q=cats+%26+dogs correctly.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.4 Statelessness

**The most misunderstood REST principle.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    STATELESSNESS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STATELESS = Every request contains ALL the information         │
│  the server needs to process it. The server does NOT            │
│  remember anything from your previous requests.                 │
│                                                                 │
│                                                                 │
│  STATEFUL (NOT how REST works):                                 │
│  ──────────────────────────────                                 │
│  Request 1: "Hi, I'm Alice."                                    │
│  Server:    "Noted. I'll remember you."                         │
│  Request 2: "Show me my orders."                                │
│  Server:    "Ah, Alice! Here are your orders."                  │
│             (Server REMEMBERS who you are from Request 1)       │
│                                                                 │
│                                                                 │
│  STATELESS (how REST works):                                    │
│  ───────────────────────────                                    │
│  Request 1: "Hi, I'm Alice. Here's my ID card."                 │
│  Server:    "Hi Alice. What do you need?"                       │
│  Request 2: "I'm Alice. Here's my ID card. Show me my orders."  │
│  Server:    "Hi Alice. Here are your orders."                   │
│             (Server does NOT remember Request 1.                │
│              You showed your ID AGAIN.)                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Library analogy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  STATEFUL LIBRARY (bad):                                        │
│  ────────────────────────                                       │
│  You: "Hi, I'm Alice."                                          │
│  Librarian: (writes "Alice" on a sticky note)                   │
│  You: "Show me my borrowed books."                              │
│  Librarian: (looks at sticky note) "Alice... here they are."    │
│                                                                 │
│  Problem: What if a DIFFERENT librarian handles your next       │
│  request? They don't have the sticky note. They don't know      │
│  who you are. The system breaks.                                │
│                                                                 │
│                                                                 │
│  STATELESS LIBRARY (REST):                                      │
│  ──────────────────────────                                     │
│  You: "Here's my library card. Show me my borrowed books."      │
│  ANY librarian: (scans card, looks up records) "Here they are." │
│                                                                 │
│  ANY librarian can handle ANY request because every request     │
│  carries everything needed. No sticky notes. No memory.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why statelessness matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY STATELESSNESS MATTERS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCALABILITY                                                    │
│  ├─ Any server can handle any request                           │
│  ├─ Add more servers easily (load balancing)                    │
│  └─ No need to route user X to server Y                         │
│                                                                 │
│  RELIABILITY                                                    │
│  ├─ If a server crashes, no state is lost                       │
│  ├─ Another server picks up the next request seamlessly         │
│  └─ No "session expired" errors                                 │
│                                                                 │
│  SIMPLICITY                                                     │
│  ├─ Each request is self-contained, independently debuggable    │
│  └─ No hidden dependencies between requests                     │
│                                                                 │
│                                                                 │
│  Connection to Async (Week 1, Lecture 3):                       │
│  Remember when we said async servers can handle thousands       │
│  of concurrent requests? Statelessness makes that possible.     │
│  If the server had to remember state for each client,           │
│  handling 10,000 clients means storing 10,000 state objects.    │
│  Stateless = no per-client memory. Pure simplicity.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "But wait — users DO log in. Shopping carts DO exist. How? The state is stored in a DATABASE or in a TOKEN (JWT) that the client sends with every request. The SERVER itself is stateless — it doesn't hold state in memory between requests. We'll implement this properly in Week 5 with JWTs."

---

## 6.5 Uniform Interface

**The principle that makes REST predictable.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   UNIFORM INTERFACE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Every REST API uses the SAME conventions:                      │
│                                                                 │
│  1. RESOURCES identified by URLs                                │
│     /users/42  (not /getUserById?id=42)                         │
│                                                                 │
│  2. METHODS indicate the action                                 │
│     GET /users/42    (read)                                     │
│     PUT /users/42    (replace)                                  │
│     DELETE /users/42 (remove)                                   │
│                                                                 │
│  3. REPRESENTATIONS are exchanged (usually JSON)                │
│     The JSON you send/receive REPRESENTS the resource           │
│     — it's not the resource itself.                             │
│                                                                 │
│  4. SELF-DESCRIPTIVE messages                                   │
│     Headers tell you everything: content type, auth, caching.   │
│     You don't need external documentation to parse a response.  │
│                                                                 │
│                                                                 │
│  The benefit: If you know ONE REST API, you can guess           │
│  how ANY REST API works. The patterns are universal.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How methods + URLs combine to form CRUD operations:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE REST CHEAT SHEET (CRUD)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CRUD Operation    HTTP Method    URL Pattern    Status Code    │
│  ──────────────    ───────────    ───────────    ───────────    │
│  Create            POST           /books         201 Created    │
│  Read (all)        GET            /books         200 OK         │
│  Read (one)        GET            /books/42      200 OK         │
│  Update (full)     PUT            /books/42      200 OK         │
│  Update (partial)  PATCH          /books/42      200 OK         │
│  Delete            DELETE         /books/42      204 No Content │
│                                                                 │
│  This pattern is UNIVERSAL across REST APIs.                    │
│  Learn it once, use it everywhere.                              │
│                                                                 │
│  Note: POST goes to the COLLECTION (/books)                     │
│        Everything else goes to the SPECIFIC RESOURCE (/books/42)│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
**The Richardson Maturity Model — How REST-Complete Is Your API?**

> "REST is not binary — you either have it or you don't. Leonard Richardson described a maturity spectrum. Most real production APIs operate at Level 2. Knowing Level 3 exists makes you a more deliberate designer."

```
┌─────────────────────────────────────────────────────────────────┐
│              RICHARDSON MATURITY MODEL                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Level 0 — The Swamp of POX                                     │
│  └─ One endpoint, POST for everything, no semantic methods      │
│     POST /api  {"action": "getBook", "id": 42}                  │
│     (RPC-style: SOAP, XML-RPC fall here)                        │
│                                                                 │
│  Level 1 — Resources                                            │
│  └─ Separate URLs per resource, but POST for all actions        │
│     POST /books/42  {"action": "get"}                           │
│     POST /books/42  {"action": "delete"}                        │
│                                                                 │
│  Level 2 — HTTP Verbs          ← THIS COURSE                    │
│  └─ Resources + correct HTTP methods + meaningful status codes  │
│     GET /books/42                                               │
│     DELETE /books/42                                            │
│     This is what most people mean by "RESTful".                 │
│                                                                 │
│  Level 3 — HATEOAS             ← Awareness only                 │
│  └─ Responses include links to next available actions           │
│     The client discovers capabilities from the response itself  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**HATEOAS — Hypermedia As The Engine Of Application State:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    LEVEL 2 vs LEVEL 3 RESPONSE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Level 2 (what you will build):                                 │
│  {                                                              │
│    "id": 42,                                                    │
│    "title": "Clean Code",                                       │
│    "available": true                                            │
│  }                                                              │
│                                                                 │
│  Level 3 HATEOAS response:                                      │
│  {                                                              │
│    "id": 42,                                                    │
│    "title": "Clean Code",                                       │
│    "available": true,                                           │
│    "_links": {                    ← available next actions       │
│      "self":   { "href": "/books/42",      "method": "GET" },   │
│      "borrow": { "href": "/books/42/loans","method": "POST" },  │
│      "author": { "href": "/authors/7",     "method": "GET" },   │
│      "delete": { "href": "/books/42",      "method": "DELETE" } │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  The client can navigate the entire API without documentation.  │
│  The response is self-describing.                               │
│                                                                 │
│  Reality: most production APIs stop at Level 2. The            │
│  implementation cost of HATEOAS rarely justifies itself for    │
│  private or partner APIs with a known client contract.          │
│  Public APIs (like GitHub's) sometimes use partial HATEOAS.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This course builds Level 2 throughout. Understanding Level 3 means you can choose to add `_links` to key responses when discoverability genuinely matters — and you can have an informed opinion when someone asks whether your API is 'truly RESTful'."

---

## 6.6 Good vs Bad API Design

**Now apply everything. Can you spot what's wrong?**

```
┌─────────────────────────────────────────────────────────────────┐
│                GOOD vs BAD URL DESIGN                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ BAD                          ✅ GOOD                        │
│  ───────                         ──────                         │
│  GET /getUser?id=42              GET /users/42                  │
│  POST /createUser                POST /users                    │
│  POST /deleteUser?id=42          DELETE /users/42               │
│  GET /getUserOrders?uid=42       GET /users/42/orders           │
│  POST /updateEmail               PATCH /users/42                │
│                                                                 │
│  WHY "BAD" IS BAD:                                              │
│  ├─ Verbs in URLs (/getUser) — the METHOD is the verb           │
│  ├─ POST for everything — methods have MEANING, use them        │
│  ├─ Inconsistent patterns — harder to learn and predict         │
│  └─ No resource hierarchy — relationships unclear               │
│                                                                 │
│  WHY "GOOD" IS GOOD:                                            │
│  ├─ URLs are NOUNS (resources)                                  │
│  ├─ Methods are VERBS (actions)                                 │
│  ├─ Consistent, predictable patterns                            │
│  └─ Hierarchy shows relationships (user → user's orders)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A complete, well-designed API surface:**

```
┌─────────────────────────────────────────────────────────────────┐
│          LIBRARY API — FULL DESIGN EXAMPLE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BOOKS:                                                         │
│  GET    /books                  List all books                  │
│  POST   /books                  Add a new book                  │
│  GET    /books/{id}             Get a specific book             │
│  PUT    /books/{id}             Replace a book's full record    │
│  PATCH  /books/{id}             Update specific book fields     │
│  DELETE /books/{id}             Remove a book                   │
│                                                                 │
│  MEMBERS:                                                       │
│  GET    /members                List all members                │
│  POST   /members                Register a new member           │
│  GET    /members/{id}           Get a member profile            │
│  PATCH  /members/{id}           Update member info              │
│  DELETE /members/{id}           Remove a member                 │
│                                                                 │
│  LOANS (nested under members):                                  │
│  GET    /members/{id}/loans     List member's active loans      │
│  POST   /members/{id}/loans     Borrow a book (create a loan)   │
│  GET    /members/{id}/loans/{lid}  Get loan details             │
│  DELETE /members/{id}/loans/{lid}  Return a book (end loan)     │
│                                                                 │
│  FILTERING & PAGINATION:                                        │
│  GET /books?author=martin&available=true&page=1&limit=20        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice the pattern. Once you see the `/books` endpoints, you can PREDICT the `/members` endpoints without documentation. That's the power of a uniform interface."
**API Versioning — Designing for Change from Day One**

> "Your API will change. You will rename fields, remove endpoints, change authentication methods. Without a versioning strategy, every breaking change forces every client to update simultaneously — or breaks them silently. This is solved with versioning, and you must choose a strategy before your first endpoint goes live."

```
┌─────────────────────────────────────────────────────────────────┐
│              THE THREE VERSIONING STRATEGIES                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STRATEGY 1: URL PATH — most common                             │
│  ────────────────────────────────                               │
│  https://api.example.com/v1/books                               │
│  https://api.example.com/v2/books                               │
│                                                                 │
│  ✅ Immediately obvious — visible in every URL and log          │
│  ✅ Testable in any browser or tool without config              │
│  ✅ Easy to route: Nginx sends /v1/* to one service,            │
│     /v2/* to another                                            │
│  ❌ Version in path is technically "not pure REST" —            │
│     the URL should identify the resource, not the API version   │
│  ❌ Clients must change every hard-coded URL on upgrade         │
│                                                                 │
│                                                                 │
│  STRATEGY 2: HEADER VERSIONING — purist approach                │
│  ─────────────────────────────────────────────                  │
│  GET /books HTTP/1.1                                            │
│  Accept: application/vnd.libraryapi.v2+json                     │
│  — or —                                                         │
│  API-Version: 2                                                 │
│                                                                 │
│  ✅ URLs remain clean resource identifiers                      │
│  ❌ Cannot test by typing a URL in a browser                    │
│  ❌ Easy to forget; wrong version returned if header missing    │
│  ❌ Harder to inspect in simple debugging tools                 │
│                                                                 │
│                                                                 │
│  STRATEGY 3: QUERY PARAMETER — simple, limited                  │
│  ─────────────────────────────────────────────                  │
│  GET /books?api_version=2                                       │
│                                                                 │
│  ✅ Easy to test — share the URL with the version included      │
│  ❌ Query params are for filtering data, not routing versions   │
│  ❌ Complicates caching (cache key includes all query params)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What the industry actually uses:**

```
┌─────────────────────────────────────────────────────────────────┐
│             REAL-WORLD VERSIONING EXAMPLES                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Stripe:   https://api.stripe.com/v1/charges   (URL)            │
│  GitHub:   https://api.github.com/repos/...    (URL + header)   │
│  Twilio:   https://api.twilio.com/2010-04-01/  (date URL)       │
│  Twitter:  https://api.twitter.com/2/tweets    (URL)            │
│                                                                 │
│  URL path versioning wins in practice by a wide margin.         │
│                                                                 │
│                                                                 │
│  BREAKING vs NON-BREAKING CHANGES:                              │
│                                                                 │
│  Safe (no version bump needed):                                 │
│  ├─ Adding a new optional response field                        │
│  ├─ Adding a new endpoint                                       │
│  └─ Adding a new optional request parameter with a default      │
│     Clients ignore fields they don't recognise. Safe.           │
│                                                                 │
│  Breaking (MUST version):                                       │
│  ├─ Removing or renaming a response field                       │
│  ├─ Changing a field's type (string → int)                      │
│  ├─ Removing an endpoint                                        │
│  └─ Changing authentication method                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How versioning looks in FastAPI (preview — full treatment Week 4):**

```python
from fastapi import FastAPI, APIRouter

app = FastAPI()

# Version 1 router — original shape
router_v1 = APIRouter(prefix="/v1")

@router_v1.get("/books/{book_id}")
async def get_book_v1(book_id: int):
    return {"id": book_id, "title": "Clean Code"}


# Version 2 router — added fields (breaking for strict clients)
router_v2 = APIRouter(prefix="/v2")

@router_v2.get("/books/{book_id}")
async def get_book_v2(book_id: int):
    return {
        "id": book_id,
        "title": "Clean Code",
        "author_id": 7,       # New field
        "edition": 1,         # New field
    }


app.include_router(router_v1)
app.include_router(router_v2)

# GET /v1/books/42  →  {"id": 42, "title": "Clean Code"}
# GET /v2/books/42  →  {"id": 42, "title": "...", "author_id": 7, ...}
# Both live simultaneously. Old clients are not broken.
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏️  PRACTICE CHECKPOINT 6.6                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q1. You want to add a new optional "isbn" field to the GET     │
│      /books/{id} response. Do you need to bump to v2? Why?      │
│                                                                 │
│  Q2. You rename the "author" field to "author_name". Is this    │
│      a breaking change? Describe a two-version migration plan.  │
│                                                                 │
│  Q3. You are building a public API for third-party developers   │
│      who build their own mobile apps. Which versioning          │
│      strategy is most appropriate, and why?                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
```
┌─────────────────────────────────────────────────────────────────┐
│                   ✅  CHECKPOINT SOLUTIONS 6.6                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A1. No. Adding a new optional field is a non-breaking,         │
│      additive change. Existing clients that do not recognise    │
│      "isbn" simply ignore it. Only removing or renaming         │
│      existing fields, or changing their types, requires a bump. │
│                                                                 │
│  A2. Yes. Clients reading "author" will receive null/undefined  │
│      after the rename — silent data loss. Migration plan:       │
│      In v2, include BOTH "author" (deprecated, same value) and  │
│      "author_name" (new). Document "author" as deprecated with  │
│      a sunset date. In v3 (or after the sunset date), remove    │
│      "author". Clients have a migration window.                 │
│                                                                 │
│  A3. URL path versioning. Third-party developers need to share  │
│      URLs in documentation, test in browsers, inspect in        │
│      network tabs, and debug with curl — all without special    │
│      header configuration. URL versioning has the lowest        │
│      friction for external developers with unknown tooling.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
## 6.7 CORS — Why Browsers Block Your Requests

> "This is one of the first walls beginners hit when connecting a frontend to a FastAPI backend. The browser blocks the request with a cryptic error. The server is running. The code looks right. But nothing works. Understanding why turns a mystery into a two-line configuration fix."

**The Same-Origin Policy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE SAME-ORIGIN POLICY                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Browsers enforce the Same-Origin Policy (SOP):                 │
│  JavaScript can only make requests to the SAME ORIGIN as the   │
│  page it is running on.                                         │
│                                                                 │
│  ORIGIN = scheme + host + port (all three must match)           │
│                                                                 │
│  https://app.example.com:443                                    │
│  ──┬───   ───────┬──────  ─┬─                                   │
│  scheme        host       port                                  │
│                                                                 │
│  SAME origin (allowed):                                         │
│  https://app.example.com/api/users    ← same scheme+host+port   │
│                                                                 │
│  DIFFERENT origin (blocked by default):                         │
│  http://app.example.com               ← scheme differs          │
│  https://api.example.com             ← host differs             │
│  https://app.example.com:8080        ← port differs             │
│  https://evil.com/data               ← completely different     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The practical problem — your dev setup is two different origins:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE TYPICAL SCENARIO                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  React dev server:   http://localhost:3000                       │
│  FastAPI backend:    http://localhost:8000                       │
│                                                                 │
│  Different ports = different origins.                           │
│  React's JavaScript tries to fetch from FastAPI.                │
│  The browser blocks it.                                         │
│                                                                 │
│  Browser console error:                                         │
│  ╔═══════════════════════════════════════════════════════════╗   │
│  ║ Access to fetch at 'http://localhost:8000/books' from     ║   │
│  ║ origin 'http://localhost:3000' has been blocked by CORS   ║   │
│  ║ policy: No 'Access-Control-Allow-Origin' header is        ║   │
│  ║ present on the requested resource.                        ║   │
│  ╚═══════════════════════════════════════════════════════════╝   │
│                                                                 │
│  CRITICAL: The request DID reach your FastAPI server.           │
│            The response DID come back.                          │
│            The BROWSER refused to give it to JavaScript.        │
│            CORS is client-side enforcement, not server-side.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**CORS — the server grants permission:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      HOW CORS WORKS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CORS = Cross-Origin Resource Sharing                           │
│                                                                 │
│  The server adds response headers to GRANT permission:          │
│  "Yes, I trust scripts running from this other origin."         │
│                                                                 │
│  Access-Control-Allow-Origin: http://localhost:3000             │
│  └─ "Only scripts from localhost:3000 may read my responses"    │
│                                                                 │
│  Access-Control-Allow-Origin: *                                 │
│  └─ "Any origin may read my responses"                          │
│     Use for fully public APIs with no credentials.              │
│     NEVER use * when allow_credentials=True.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Two types of CORS requests — simple vs preflighted:**

```
┌─────────────────────────────────────────────────────────────────┐
│                SIMPLE vs PREFLIGHTED REQUESTS                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SIMPLE REQUEST (no preflight needed):                          │
│  ├─ Method: GET, HEAD, or POST only                             │
│  └─ Only basic, form-standard headers                           │
│                                                                 │
│  Browser → request → Server                                     │
│  Server  ← response with Access-Control-Allow-Origin ← checks  │
│  ✅ browser grants response to JS                               │
│                                                                 │
│                                                                 │
│  PREFLIGHTED REQUEST (browser asks permission first):           │
│  ├─ Methods: PUT, PATCH, DELETE                                 │
│  ├─ Custom headers: Authorization, Content-Type: application/   │
│  │   json (anything beyond basic form headers)                  │
│  └─ Triggers an automatic OPTIONS preflight before the request  │
│                                                                 │
│  STEP 1 — PREFLIGHT (automatic, you do not write this):         │
│  Browser ──OPTIONS /books──▶ Server                             │
│  Origin: http://localhost:3000                                  │
│  Access-Control-Request-Method: POST                            │
│  Access-Control-Request-Headers: Content-Type, Authorization    │
│                                                                 │
│  Server ◀── must respond ──                                     │
│  Access-Control-Allow-Origin: http://localhost:3000             │
│  Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE    │
│  Access-Control-Allow-Headers: Content-Type, Authorization      │
│                                                                 │
│  STEP 2 — ACTUAL REQUEST (only if preflight approved):          │
│  Browser ──POST /books──▶ Server                                │
│                                                                 │
│  If your server returns 404 or 405 on OPTIONS, the preflight    │
│  fails and the actual request is never sent.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**FastAPI CORSMiddleware — what you will add in Week 2:**

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",        # React dev server
        "https://app.yourdomain.com",   # Production frontend
    ],
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    allow_credentials=True,             # Allow cookies cross-origin
)
```

```
┌─────────────────────────────────────────────────────────────────┐
│              CORSMIDDLEWARE SETTINGS EXPLAINED                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  allow_origins=[...]                                            │
│  └─ Which frontend origins may call this API                    │
│     ["*"] = any origin. Only for APIs with no auth/cookies.     │
│     ⚠️ ["*"] with allow_credentials=True is INVALID —           │
│     browsers reject this combination outright.                  │
│                                                                 │
│  allow_methods=[...]                                            │
│  └─ Which HTTP methods are permitted cross-origin               │
│                                                                 │
│  allow_headers=[...]                                            │
│  └─ Which request headers the frontend JavaScript may send      │
│     Must include "Authorization" for Bearer token APIs          │
│                                                                 │
│  allow_credentials=True                                         │
│  └─ Whether browsers may send cookies cross-origin              │
│     Required for cookie-based authentication across origins     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The most important misconception about CORS:**

```
┌─────────────────────────────────────────────────────────────────┐
│                CORS DOES NOT PROTECT YOUR SERVER                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CORS protects the BROWSER USER.                                │
│                                                                 │
│  curl, Postman, httpx — no Same-Origin Policy. They always      │
│  reach your server regardless of CORS configuration.            │
│                                                                 │
│  curl https://api.example.com/data     ← always works           │
│  Postman → your API                    ← always works           │
│  JS on evil.com → your API             ← CORS blocks this       │
│                                                                 │
│  The protection prevents evil.com's JavaScript from silently    │
│  making authenticated requests to your API using a victim       │
│  user's stored cookies or credentials without their knowledge.  │
│                                                                 │
│  This is why your unit tests and curl debugging always work,    │
│  but the browser refuses. It is not a server bug.               │
│  It is the browser protecting its user.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏️  PRACTICE CHECKPOINT 6.7                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q1. React on localhost:3000 calls FastAPI on localhost:8000.   │
│      Are these the same origin? What must you configure?        │
│                                                                 │
│  Q2. A student says: "I added CORSMiddleware but my Postman     │
│      request still works while my browser request still fails." │
│      Is CORSMiddleware broken? What is the actual problem?      │
│                                                                 │
│  Q3. You set allow_origins=["*"] and allow_credentials=True     │
│      together. What happens, and why does the browser reject it? │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
```
┌─────────────────────────────────────────────────────────────────┐
│                   ✅  CHECKPOINT SOLUTIONS 6.7                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A1. No — different ports mean different origins. You must add  │
│      CORSMiddleware with                                        │
│      allow_origins=["http://localhost:3000"].                   │
│                                                                 │
│  A2. CORSMiddleware is not broken. Postman does not enforce the │
│      Same-Origin Policy — it is not a browser. CORS is enforced │
│      exclusively by browsers. If the browser still fails, check │
│      whether the exact origin string in allow_origins matches   │
│      the frontend's origin precisely (scheme, host, and port).  │
│                                                                 │
│  A3. The browser rejects this combination entirely. Allowing    │
│      all origins (*) with credentials would permit ANY website  │
│      to make authenticated requests using a user's stored       │
│      cookies, completely defeating the security model.          │
│      The spec explicitly forbids it. If you need credentials,   │
│      you must list specific trusted origins explicitly.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 7: JSON — THE DATA FORMAT

## 7.1 Why JSON Won

```
┌─────────────────────────────────────────────────────────────────┐
│                      WHY JSON?                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  APIs need a format to send data as text. The two contenders:   │
│                                                                 │
│  XML (old):                                                     │
│  <book>                                                         │
│    <title>Clean Code</title>                                    │
│    <author>Robert Martin</author>                               │
│    <available>true</available>                                  │
│  </book>                                                        │
│                                                                 │
│  JSON (modern):                                                 │
│  {                                                              │
│    "title": "Clean Code",                                       │
│    "author": "Robert Martin",                                   │
│    "available": true                                            │
│  }                                                              │
│                                                                 │
│  JSON won because:                                              │
│  ├─ Lighter — less boilerplate, smaller messages                │
│  ├─ Readable — humans can parse it at a glance                  │
│  ├─ Native to JavaScript — web's lingua franca                  │
│  ├─ Maps directly to Python dicts and lists — no conversion     │
│  └─ Almost every language has built-in support                  │
│                                                                 │
│  JSON stands for JavaScript Object Notation.                    │
│  But don't let the name fool you — every language uses it.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.2 JSON Syntax Rules

```
┌─────────────────────────────────────────────────────────────────┐
│                   JSON SYNTAX RULES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SIX DATA TYPES:                                                │
│                                                                 │
│  String:    "hello"             (always double quotes!)         │
│  Number:    42, 3.14, -7        (no quotes, no NaN/Infinity)    │
│  Boolean:   true, false         (lowercase! not True/False)     │
│  Null:      null                (lowercase! not None)           │
│  Array:     [1, 2, 3]           (ordered list of values)        │
│  Object:    {"key": "value"}    (key-value pairs)               │
│                                                                 │
│                                                                 │
│  STRICT RULES:                                                  │
│  ├─ Keys MUST be strings in double quotes                       │
│  │    ✅ {"name": "Alice"}                                      │
│  │    ❌ {name: "Alice"}        (no quotes on key)              │
│  │    ❌ {'name': 'Alice'}      (single quotes)                 │
│  │                                                              │
│  ├─ No trailing commas                                          │
│  │    ✅ {"a": 1, "b": 2}                                       │
│  │    ❌ {"a": 1, "b": 2,}     (trailing comma)                 │
│  │                                                              │
│  ├─ No comments                                                 │
│  │    ❌ {"a": 1}  // this is a comment                         │
│  │                                                              │
│  └─ No undefined, no functions, no dates (just strings)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A realistic JSON response:**

```json
{
  "id": 42,
  "title": "Clean Code",
  "author": {
    "name": "Robert Martin",
    "website": "https://cleancoder.com"
  },
  "tags": ["programming", "best-practices", "software"],
  "available": true,
  "rating": 4.7,
  "borrowed_by": null
}
```

**Notice:** Nested objects, arrays, multiple types, null — all in one structure. This maps DIRECTLY to Python.

---

## 7.3 Python ↔ JSON (The json Module)

**Python has built-in JSON support. No pip install needed.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 PYTHON ↔ JSON TYPE MAPPING                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Python              JSON              Direction                │
│  ──────              ────              ─────────                │
│  dict           →    object {}         ↔                        │
│  list, tuple    →    array []          ↔ (always returns list)  │
│  str            →    string ""         ↔                        │
│  int, float     →    number            ↔                        │
│  True           →    true              ↔                        │
│  False          →    false             ↔                        │
│  None           →    null              ↔                        │
│                                                                 │
│  ⚠️ WATCH THE DIFFERENCES:                                      │
│  Python: True, False, None    (capitalized, None not null)      │
│  JSON:   true, false, null    (lowercase)                       │
│  The json module handles this conversion automatically.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The two core operations:**

```python
import json

# ═══════════════════════════════════════
# SERIALIZATION: Python → JSON string
# "Convert my Python data into text that can travel over HTTP"
# ═══════════════════════════════════════

book: dict = {
    "title": "Clean Code",
    "author": "Robert Martin",
    "available": True,    # Python True
    "borrowed_by": None,  # Python None
    "tags": ["programming", "best-practices"]
}

json_string: str = json.dumps(book, indent=2)
print(json_string)
```

Output:
```json
{
  "title": "Clean Code",
  "author": "Robert Martin",
  "available": true,
  "borrowed_by": null,
  "tags": [
    "programming",
    "best-practices"
  ]
}
```

```python
# ═══════════════════════════════════════
# DESERIALIZATION: JSON string → Python
# "Convert the text I received from HTTP into Python objects"
# ═══════════════════════════════════════

received: str = '{"id": 42, "title": "Clean Code", "available": true}'

data: dict = json.loads(received)
print(data)
# {'id': 42, 'title': 'Clean Code', 'available': True}

print(type(data["available"]))
# <class 'bool'> — json.loads converted "true" → True automatically
```

```
┌─────────────────────────────────────────────────────────────────┐
│              THE TWO JSON FUNCTIONS YOU NEED                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  json.dumps(python_object) → JSON string                        │
│       │  │                                                      │
│       │  └─ "s" = string (output is a string)                   │
│       └─ "dump" = serialize                                     │
│                                                                 │
│  json.loads(json_string)   → Python object                      │
│       │  │                                                      │
│       │  └─ "s" = string (input is a string)                    │
│       └─ "load" = deserialize                                   │
│                                                                 │
│  Also:                                                          │
│  json.dump(obj, file)  → write to file                          │
│  json.load(file)       → read from file                         │
│                                                                 │
│  Mnemonic: The "s" versions work with Strings.                  │
│            The non-"s" versions work with files.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.4 Dataclasses to JSON (Connection to Week 1)

**Connection to what you've learned:**

> "Remember dataclasses from Week 1, Lecture 2? Here's WHY you learned them. A dataclass defines the SHAPE of structured data — the same shape as a JSON object."

```python
from dataclasses import dataclass, asdict
from typing import Optional
import json

@dataclass
class Author:
    name: str
    website: Optional[str] = None

@dataclass
class Book:
    id: int
    title: str
    author: Author
    available: bool = True
    tags: list[str] = None

    def __post_init__(self):
        if self.tags is None:
            self.tags = []

# Create a Book instance (Python world)
book = Book(
    id=42,
    title="Clean Code",
    author=Author(name="Robert Martin", website="https://cleancoder.com"),
    tags=["programming", "best-practices"]
)

# Dataclass → dict → JSON string (serialization pipeline)
book_dict: dict = asdict(book)
json_string: str = json.dumps(book_dict, indent=2)
print(json_string)
```

Output:
```json
{
  "id": 42,
  "title": "Clean Code",
  "author": {
    "name": "Robert Martin",
    "website": "https://cleancoder.com"
  },
  "available": true,
  "tags": [
    "programming",
    "best-practices"
  ]
}
```

**Deserialization — JSON back to a dataclass:**

```python
# JSON string → dict → Dataclass (deserialization pipeline)
received_json: str = '''
{
  "id": 99,
  "title": "Design Patterns",
  "author": {"name": "Gang of Four", "website": null},
  "available": true,
  "tags": ["programming"]
}
'''

data: dict = json.loads(received_json)

# Manual reconstruction (nested objects need manual handling)
book = Book(
    id=data["id"],
    title=data["title"],
    author=Author(**data["author"]),   # Unpack nested dict
    available=data["available"],
    tags=data["tags"]
)

print(book)
# Book(id=99, title='Design Patterns', author=Author(name='Gang of Four', ...
```

**Notice the pain:**

> "Deserializing nested dataclasses is MANUAL and tedious. You have to unpack every nested dict yourself. What if a field is missing? What if the type is wrong? What if `author` is `null`? You'd need try/except everywhere."

```
┌─────────────────────────────────────────────────────────────────┐
│                THE PAIN POINT (ON PURPOSE)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Dataclass + json.loads() deserialization problems:             │
│                                                                 │
│  ├─ No automatic validation (age = -5 just works)               │
│  ├─ No type coercion ("42" stays a string, not int)             │
│  ├─ Nested objects need manual unpacking                        │
│  ├─ Missing fields cause KeyError (not a helpful message)       │
│  └─ No way to define constraints (min, max, regex)              │
│                                                                 │
│  This is EXACTLY the problem that Pydantic solves.              │
│  Week 2, Lecture 3 will introduce Pydantic BaseModel —          │
│  think of it as "dataclass + validation + serialization"        │
│  built specifically for this JSON ↔ Python conversion.          │
│                                                                 │
│  dataclass = foundation                                         │
│  Pydantic  = dataclass on steroids for HTTP/API work            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.5 The Serialization Pipeline (Preview of Pydantic)

**The full journey of data in an API:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE SERIALIZATION PIPELINE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT SENDS REQUEST:                                          │
│                                                                 │
│  JSON string ──▶ Python dict ──▶ Validated object               │
│  (over HTTP)     (json.loads)    (Pydantic, Week 2 Lecture 3)   │
│                                                                 │
│  {"name": "Alice", "age": 30}                                   │
│        │                                                        │
│        ▼                                                        │
│  {"name": "Alice", "age": 30}   ← Python dict                   │
│        │                                                        │
│        ▼                                                        │
│  User(name="Alice", age=30)     ← Validated, typed object       │
│                                                                 │
│                                                                 │
│  SERVER SENDS RESPONSE:                                         │
│                                                                 │
│  Python object ──▶ Python dict ──▶ JSON string                  │
│  (your data)       (asdict)        (json.dumps, over HTTP)      │
│                                                                 │
│  User(name="Alice", age=30)                                     │
│        │                                                        │
│        ▼                                                        │
│  {"name": "Alice", "age": 30}   ← Python dict                   │
│        │                                                        │
│        ▼                                                        │
│  '{"name": "Alice", "age": 30}' ← JSON string (sent to client)  │
│                                                                 │
│                                                                 │
│  FastAPI + Pydantic handle this ENTIRE pipeline automatically.  │
│  But now you understand what's happening under the hood.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Building the pipeline by hand to understand it:**

```python
import json
from dataclasses import dataclass, asdict
from typing import Optional


@dataclass
class CreateBookRequest:
    """What the client sends (incoming data)"""
    title: str
    author: str
    year: Optional[int] = None


@dataclass
class BookResponse:
    """What we send back (outgoing data)"""
    id: int
    title: str
    author: str
    year: Optional[int]
    available: bool


def handle_create_book(raw_json: str) -> str:
    """
    Simulate the full request → response pipeline.
    
    In FastAPI, this entire function is automatic.
    We do it manually here so you understand every step.
    """
    
    # STEP 1: Deserialize — JSON string → Python dict
    data: dict = json.loads(raw_json)
    
    # STEP 2: Validate — dict → typed object (manual, painful)
    if "title" not in data or "author" not in data:
        error = {"error": "Missing required fields", "status": 400}
        return json.dumps(error)
    
    request = CreateBookRequest(
        title=data["title"],
        author=data["author"],
        year=data.get("year"),  # Optional — might not exist
    )
    
    # STEP 3: Business logic — process the request
    # (In real life: save to database, generate ID)
    new_id: int = 99  # Simulated
    
    response = BookResponse(
        id=new_id,
        title=request.title,
        author=request.author,
        year=request.year,
        available=True,
    )
    
    # STEP 4: Serialize — Python object → dict → JSON string
    response_dict: dict = asdict(response)
    return json.dumps(response_dict, indent=2)


# Simulate a client request
client_json: str = '{"title": "Clean Code", "author": "Robert Martin", "year": 2008}'
response_json: str = handle_create_book(client_json)
print(response_json)
```

Output:
```json
{
  "id": 99,
  "title": "Clean Code",
  "author": "Robert Martin",
  "year": 2008,
  "available": true
}
```

> "That's 40 lines to handle ONE endpoint. The validation is fragile. The type checking is nonexistent. Now imagine doing this for 50 endpoints. This is why FastAPI + Pydantic exist — they automate this entire pipeline with validation, type coercion, error messages, and documentation. You'll see this next lecture."
---
## 7.6 File Uploads — When JSON Isn't Enough

> "JSON is perfect for structured data: strings, numbers, booleans, nested objects. But JSON cannot carry binary data — an image, a PDF, a video file — without corrupting it or wrapping it in base64 encoding that bloats its size by 33%. For binary files, HTTP uses a different Content-Type entirely: `multipart/form-data`."

**What multipart/form-data is:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   MULTIPART/FORM-DATA                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A multipart request body is divided into PARTS.                │
│  Each part is separated by a BOUNDARY string.                   │
│  Each part can have its own Content-Type.                        │
│                                                                 │
│  This is the mechanism HTML <input type="file"> forms use.      │
│  It is also how your API will receive file uploads.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What a raw multipart request looks like:**

```
POST /books/42/cover HTTP/1.1
Host: api.library.com
Authorization: Bearer abc123
Content-Type: multipart/form-data; boundary=----Boundary7MA4YWxk

------Boundary7MA4YWxk
Content-Disposition: form-data; name="caption"

First edition cover
------Boundary7MA4YWxk
Content-Disposition: form-data; name="cover_image"; filename="cover.jpg"
Content-Type: image/jpeg

[binary JPEG bytes here — not human-readable]
------Boundary7MA4YWxk--
```

**Dissecting the format:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    MULTIPART ANATOMY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Content-Type: multipart/form-data; boundary=BOUNDARY_STRING    │
│  └─ The boundary is a unique string that will not appear in     │
│     any of the content. Chosen randomly by the client.          │
│                                                                 │
│  Each part structure:                                           │
│  --BOUNDARY                 ← two dashes + boundary = part start│
│  Content-Disposition: form-data; name="field_name"              │
│     For files: name="upload"; filename="photo.jpg"              │
│  Content-Type: image/jpeg   ← optional for files                │
│  [blank line]                                                   │
│  [part content — text or binary]                                │
│                                                                 │
│  End marker:                                                    │
│  --BOUNDARY--               ← two dashes on both sides = done   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Sending file uploads with curl and Python:**

```bash
# Upload a single file — curl sets multipart automatically with -F
curl -X POST https://api.library.com/books/42/cover \
  -H "Authorization: Bearer your-token" \
  -F "cover_image=@/path/to/cover.jpg"

# Upload file + text field together
curl -X POST https://api.library.com/books/42/cover \
  -H "Authorization: Bearer your-token" \
  -F "caption=First edition cover" \
  -F "cover_image=@/path/to/cover.jpg;type=image/jpeg"
```

```python
import httpx

async def upload_book_cover(book_id: int, image_path: str) -> dict:
    """Upload a cover image. httpx sets Content-Type: multipart automatically."""

    async with httpx.AsyncClient() as client:
        with open(image_path, "rb") as image_file:
            response = await client.post(
                f"https://api.library.com/books/{book_id}/cover",
                headers={"Authorization": "Bearer abc123"},
                files={
                    # tuple: (filename, file_object, content_type)
                    "cover_image": ("cover.jpg", image_file, "image/jpeg")
                },
                data={
                    "caption": "First edition cover"  # Regular form field
                }
            )

        response.raise_for_status()
        return response.json()
```

**JSON vs multipart — when to use which:**

```
┌─────────────────────────────────────────────────────────────────┐
│              JSON vs MULTIPART — DECISION GUIDE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Use JSON (application/json) when:                              │
│  ├─ All data is structured text, numbers, booleans              │
│  └─ No binary file content                                      │
│                                                                 │
│  Use multipart/form-data when:                                  │
│  ├─ Uploading binary files (images, PDFs, videos, archives)     │
│  ├─ Uploading files AND metadata in a single request            │
│  └─ Receiving uploads from HTML <form> elements                 │
│                                                                 │
│  Can you mix JSON and file in one request?                      │
│  No. A single request body is EITHER JSON OR multipart.         │
│                                                                 │
│  For "upload file + send structured metadata":                  │
│  Option A: Use multipart — send metadata as form-data fields    │
│  Option B: Two-step — POST JSON to create the entity, get       │
│            back the ID, then POST multipart to upload the file  │
│            (cleaner separation of concerns)                     │
│  Option C: Base64-encode the file and embed in JSON             │
│            (works for small files; ~33% size penalty; awkward)  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**FastAPI handles this with UploadFile (preview — Week 3, Lecture 2):**

```python
# This is what you will write next lecture.
# Read it now so you understand the HTTP mechanism it abstracts.

from fastapi import FastAPI, File, UploadFile, Form
import shutil

app = FastAPI()

@app.post("/books/{book_id}/cover")
async def upload_cover(
    book_id: int,
    cover_image: UploadFile = File(...),   # multipart file part
    caption: str = Form(""),               # multipart text part
):
    # FastAPI already parsed the multipart body for you
    with open(f"covers/{book_id}.jpg", "wb") as f:
        shutil.copyfileobj(cover_image.file, f)

    return {
        "book_id": book_id,
        "filename": cover_image.filename,
        "content_type": cover_image.content_type,
        "caption": caption,
    }
```

> "The boundary parsing, part assembly, and binary reading are handled entirely by FastAPI. What you are learning now is what is inside the HTTP request before FastAPI touches it — so that when something breaks, you know where to look."

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏️  PRACTICE CHECKPOINT 7.6                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q1. A client sends Content-Type: application/json with a raw   │
│      JPEG binary in the body. What status code should your      │
│      server return, and why?                                    │
│                                                                 │
│  Q2. A user wants to create a new book entry AND upload the     │
│      cover image in a single request. Your endpoint currently   │
│      only accepts JSON. Name two valid redesign options.        │
│                                                                 │
│  Q3. The boundary string in a multipart request is chosen to    │
│      be long and random. What breaks if the boundary string     │
│      accidentally appears inside the binary file data?          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

▼ SOLUTIONS ──────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────┐
│                   ✅  CHECKPOINT SOLUTIONS 7.6                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A1. 415 Unsupported Media Type. The endpoint understands the   │
│      format (multipart/form-data) but the client sent the wrong │
│      Content-Type. The JPEG is valid binary — the declared type │
│      is what is wrong. Not 400 (which implies malformed input). │
│                                                                 │
│  A2. Option A: Change the endpoint to accept multipart/form-    │
│      data. Send metadata (title, author) as form-data fields    │
│      alongside the cover_image file part.                       │
│      Option B: Two-step upload. POST JSON to POST /books to     │
│      create the book and receive its ID. Then POST multipart    │
│      to POST /books/{id}/cover with just the image.             │
│                                                                 │
│  A3. The multipart parser encounters the boundary string mid-   │
│      file and interprets it as a part separator. It incorrectly │
│      ends the current part early and begins a new one. The file │
│      data is split and corrupted. This is why boundary strings  │
│      are generated to be long and random enough that the        │
│      probability of collision with any binary content is        │
│      effectively zero.                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                  HTTP QUICK REFERENCE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REQUEST FORMAT:                                                │
│      METHOD /path HTTP/1.1                                      │
│      Header-Name: header-value                                  │
│      Content-Type: application/json                             │
│                                                                 │
│      {"request": "body"}                                        │
│                                                                 │
│  RESPONSE FORMAT:                                               │
│      HTTP/1.1 STATUS_CODE Reason                                │
│      Content-Type: application/json                             │
│                                                                 │
│      {"response": "body"}                                       │
│                                                                 │
│  METHODS:                                                       │
│      GET     /items        Read all (list)                      │
│      GET     /items/42     Read one                             │
│      POST    /items        Create new → 201                     │
│      PUT     /items/42     Replace entire → 200                 │
│      PATCH   /items/42     Update partial → 200                 │
│      DELETE  /items/42     Remove → 204                         │
│                                                                 │
│  STATUS CODES:                                                  │
│      200 OK                Success, here's the data             │
│      201 Created           New resource created                 │
│      204 No Content        Success, empty body                  │
│      400 Bad Request       Malformed request                    │
│      401 Unauthorized      Missing/invalid credentials          │
│      403 Forbidden         Valid credentials, no permission     │
│      404 Not Found         Resource doesn't exist               │
│      422 Unprocessable     Valid format, invalid values         │
│      500 Internal Error    Server bug (unhandled exception)     │
│                                                                 │
│  ESSENTIAL HEADERS:                                             │
│      Content-Type: application/json   (body format)             │
│      Accept: application/json         (wanted format)           │
│      Authorization: Bearer <token>    (identity)                │
│      Host: api.example.com            (target server)           │
│                                                                 │
│  JSON ↔ PYTHON:                                                 │
│      json.dumps(obj)  → Python to JSON string                   │
│      json.loads(str)  → JSON string to Python                   │
│                                                                 │
│  CURL:                                                          │
│      curl -v URL                      Verbose GET               │
│      curl -X POST URL -H "..." -d '...'  POST with body         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  HTTP = A TEXT CONVERSATION WITH RULES                          │
│                                                                 │
│  A client sends a TEXT message (request).                       │
│  A server sends a TEXT message back (response).                 │
│  Both messages have the same structure:                         │
│  first line, headers, blank line, body.                         │
│                                                                 │
│  ┌──────────┐   request (text)    ┌──────────┐                  │
│  │  CLIENT  │ ──────────────────▶ │  SERVER  │                  │
│  │          │                     │          │                  │
│  │          │ ◀────────────────── │          │                  │
│  └──────────┘   response (text)   └──────────┘                  │
│                                                                 │
│                                                                 │
│  THE LIBRARY ANALOGY:                                           │
│  ├─ You (client) fill out a request form                        │
│  │   → METHOD says what you want to DO                          │
│  │   → URL says WHICH resource                                  │
│  │   → Headers are metadata on the form                         │
│  │   → Body is the content you're submitting                    │
│  │                                                              │
│  ├─ Librarian (server) hands back a response                    │
│  │   → Status code says WHAT HAPPENED                           │
│  │   → Headers describe the response format                     │
│  │   → Body is the data you asked for (as JSON)                 │
│  │                                                              │
│  └─ Stateless: show your card EVERY time.                       │
│     No visit remembers the last.                                │
│                                                                 │
│                                                                 │
│  REST is the principle:                                         │
│  Resources (nouns) + Methods (verbs) + Status Codes (outcomes)  │
│  = A predictable, universal API interface.                      │
│                                                                 │
│  JSON is the language:                                          │
│  The text format both sides agree to use in the body.           │
│  It maps directly to Python dicts, lists, and primitives.       │
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
│  WEEK 2, LECTURE 2 — FastAPI Fundamentals (NEXT):               │
│  └─ Everything you learned today becomes code.                  │
│     @app.get = a GET endpoint. Path params = URL variables.     │
│     Return a dict = automatically serialized to JSON.           │
│     You'll finally understand WHAT each decorator means.        │
│                                                                 │
│  WEEK 2, LECTURE 3 — Pydantic Deep Dive:                        │
│  └─ Remember the painful manual JSON ↔ dataclass conversion?    │
│     Pydantic makes it automatic, with validation, type          │
│     coercion, error messages, and serialization control.        │
│     The pain you felt in Part 7 is why Pydantic exists.         │
│                                                                 │
│  WEEK 2, LECTURE 4 — Error Handling & Dependencies:             │
│  └─ HTTPException maps DIRECTLY to status codes.                │
│     raise HTTPException(status_code=404, detail="Not found")    │
│     Your exception hierarchy knowledge (Week 1) + status        │
│     code knowledge (today) = FastAPI error handling.            │
│                                                                 │
│  WEEK 2, LECTURE 5 — Testing FastAPI:                           │
│  └─ You'll test that GET returns 200, POST returns 201,         │
│     bad input returns 422. Every status code becomes            │
│     a test assertion.                                           │
│                                                                 │
│  WEEK 4 — External APIs:                                        │
│  └─ You'll be the CLIENT, not the server.                       │
│     httpx.AsyncClient sends the exact same HTTP requests        │
│     you saw in raw text today. Methods, headers, JSON bodies    │
│     — all the same concepts, from the other side.               │
│                                                                 │
│  WEEK 5 — Authentication:                                       │
│  └─ The Authorization header you saw today becomes JWT tokens,  │
│     OAuth flows, and 401/403 responses you'll implement.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---