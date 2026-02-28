# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PAIN FIRST, PROTOCOL SECOND                                    │
│  ───────────────────────────                                    │
│  Students will BUILD the bad solution (polling) before seeing   │
│  the good one. They must feel the waste before wanting the fix. │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  HTTP = text messages. WebSocket = phone call.                  │
│  Every concept maps to this. Consistent throughout.             │
│                                                                 │
│  ASYNC PAYOFF LECTURE                                           │
│  ────────────────────                                           │
│  Week 1's async lesson was the seed. This is the harvest.       │
│  Students will see WHY async matters — a while True loop that   │
│  handles 10,000 connections because of a single `await`.        │
│  Don't re-teach async. Make them feel the reward.               │
│                                                                 │
│  BUILD ON THE STACK, DON'T REPEAT IT                            │
│  ────────────────────────────────────                           │
│  HTTP knowledge (Week 3) → WebSocket starts as HTTP             │
│  FastAPI routes (Week 3) → @app.websocket is the same idea      │
│  Depends() (Week 3) → Works in WebSocket handlers               │
│  JWT auth (Week 9) → Reuse verify_token for WebSocket           │
│  Redis pub/sub (Week 11) → Same concept, different transport    │
│  Students have 11 weeks of tools. Reference, don't re-teach.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                   WEBSOCKET FUNDAMENTALS                        │
│                     (3-4 Hour Lecture)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (40 min)                                   │
│  ├─ 1.1 The Stale Dashboard (Demonstration)                     │
│  ├─ 1.2 HTTP's Fundamental Limitation                           │
│  ├─ 1.3 The Polling Workaround (and Why It Fails)               │
│  └─ 1.4 The Phone Call Analogy                                  │
│                                                                 │
│  PART 2: THE PROTOCOL (40 min)                                  │
│  ├─ 2.1 The Handshake (HTTP → WebSocket Upgrade)                │
│  ├─ 2.2 The WebSocket Lifecycle (Open, Message, Close)          │
│  ├─ 2.3 Messages, Not Requests (A Different Mental Model)       │
│  └─ 2.4 When to Use: WebSocket vs HTTP vs SSE                   │
│                                                                 │
│  PART 3: FASTAPI WEBSOCKET ENDPOINTS (55 min)                   │
│  ├─ 3.1 @app.websocket — Your First WebSocket Route             │
│  ├─ 3.2 The WebSocket Object (accept, send, receive, close)     │
│  ├─ 3.3 The Receive Loop (The Pattern That Changes Everything)  │
│  ├─ 3.4 Handling Disconnections (WebSocketDisconnect)           │
│  └─ 3.5 Structured Messages with JSON                           │
│                                                                 │
│  PART 4: CONNECTION MANAGEMENT (50 min)                         │
│  ├─ 4.1 The Problem: Isolated Handlers                          │
│  ├─ 4.2 Building a ConnectionManager                            │
│  ├─ 4.3 Broadcasting to All Clients                             │
│  └─ 4.4 Rooms and Channels (Targeted Broadcasting)              │
│                                                                 │
│  PART 5: WEBSOCKET AUTHENTICATION (35 min)                      │
│  ├─ 5.1 The Auth Challenge (No Headers After Handshake)         │
│  ├─ 5.2 Strategy 1: Token in Query Parameter                    │
│  ├─ 5.3 Strategy 2: Token in First Message                      │
│  └─ 5.4 Choosing a Strategy (Tradeoffs)                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Stale Dashboard

**Set the scene. Use their own project.**

> "Your Task Manager is live. Two team members — Alice and Bob — are looking at the same project board. Alice marks a task as 'Done.' She sees it update. Bob sees... nothing. His board is stale. He's looking at a lie."

> "How does Bob find out? He refreshes the page. Or he waits. Or he messages Alice on Slack: 'Hey, did you finish that task?' This is absurd. You built an entire real-time backend stack — async, caching, background jobs — and the user still has to hit F5."

> "Today we fix this."

**Demonstrate the problem with code they understand:**

```python
# server.py — A simple FastAPI task board (using your existing knowledge)
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

# Shared state (imagine this is your database)
tasks: dict[int, dict] = {
    1: {"id": 1, "title": "Design API schema", "status": "in_progress"},
    2: {"id": 2, "title": "Write tests", "status": "todo"},
    3: {"id": 3, "title": "Deploy to cloud", "status": "todo"},
}
board_version: int = 0


class TaskUpdate(BaseModel):
    status: str


@app.get("/tasks")
async def get_tasks():
    return {"version": board_version, "tasks": list(tasks.values())}


@app.get("/tasks/version")
async def get_version():
    """Lightweight endpoint — just returns the version number."""
    return {"version": board_version}


@app.patch("/tasks/{task_id}")
async def update_task(task_id: int, update: TaskUpdate):
    global board_version
    tasks[task_id]["status"] = update.status
    board_version += 1
    return tasks[task_id]
```

> "Nothing new here. Standard FastAPI, standard Pydantic. You've written this in your sleep since Week 3."

> "Now here's the question: Bob's browser loaded the task board. How does it know when Alice updates a task?"

---

## 1.2 HTTP's Fundamental Limitation

**This is the root of the problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│                HTTP = ONE-WAY INITIATION                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│    CLIENT                              SERVER                   │
│    ──────                              ──────                   │
│       │                                   │                     │
│       │ ─── "Give me the tasks" ────────▶ │                     │
│       │                                   │                     │
│       │ ◀── "Here are the tasks" ──────── │                     │
│       │                                   │                     │
│       │           (silence)               │                     │
│       │                                   │                     │
│       │  Alice updates a task on server ──│                     │
│       │                                   │                     │
│       │           (silence)               │                     │
│       │                                   │  ← Server CANNOT    │
│       │    Bob has NO IDEA.               │    reach out to Bob │
│       │    The connection is GONE.        │                     │
│       │                                   │                     │
│                                                                 │
│   HTTP connections are DISPOSABLE. Send request, get response,  │
│   connection closed. The server has NO WAY to push data to a    │
│   client that isn't actively asking.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "You learned in Week 3 that HTTP is request-response. The client always initiates. The server only ever *replies*. It cannot *reach out*. Once the response is sent, the server forgets the client exists."

> "This is fine for most APIs. You ask for data, you get data. But what about when the SERVER has new information and needs to tell the CLIENT? HTTP has no mechanism for this."

---

## 1.3 The Polling Workaround (and Why It Fails)

**The obvious solution — and its cost:**

> "With your current toolkit, there's only one way to get updates: keep asking."

```python
# poll_client.py — Bob's browser, simulated
import asyncio
import httpx
import time

async def poll_for_updates():
    """Bob's client: check for updates every 2 seconds."""
    last_known_version = 0
    request_count = 0
    wasted_count = 0

    async with httpx.AsyncClient() as client:
        start = time.time()

        while time.time() - start < 30:  # Run for 30 seconds
            request_count += 1
            response = await client.get(
                "http://localhost:8000/tasks/version"
            )
            current_version = response.json()["version"]

            if current_version > last_known_version:
                print(
                    f"  [Request #{request_count}] "
                    f"NEW UPDATE! Version: {current_version}"
                )
                last_known_version = current_version
            else:
                wasted_count += 1
                print(
                    f"  [Request #{request_count}] "
                    f"No changes. (wasted request)"
                )

            await asyncio.sleep(2)

    print(f"\n  Total requests: {request_count}")
    print(f"  Wasted requests: {wasted_count}")
    print(
        f"  Efficiency: "
        f"{((request_count - wasted_count) / request_count) * 100:.0f}%"
    )

asyncio.run(poll_for_updates())
```

**Run it. Let the class watch the waste pile up.**

```
  [Request #1] No changes. (wasted request)
  [Request #2] No changes. (wasted request)
  [Request #3] No changes. (wasted request)
  [Request #4] No changes. (wasted request)
  [Request #5] No changes. (wasted request)
  [Request #6] NEW UPDATE! Version: 1
  [Request #7] No changes. (wasted request)
  [Request #8] No changes. (wasted request)
  [Request #9] No changes. (wasted request)
  [Request #10] No changes. (wasted request)
  [Request #11] No changes. (wasted request)
  [Request #12] No changes. (wasted request)
  [Request #13] No changes. (wasted request)
  [Request #14] No changes. (wasted request)
  [Request #15] NEW UPDATE! Version: 2

  Total requests: 15
  Wasted requests: 13
  Efficiency: 13%
```

**Now ask the class:**

> "13 out of 15 requests were wasted. 87% of our traffic is garbage. And that's ONE client. What happens with 1,000 users polling every 2 seconds?"

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE COST OF POLLING                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1 client, polling every 2 seconds:                             │
│  └─ 30 requests/minute                                          │
│                                                                 │
│  100 clients:                                                   │
│  └─ 3,000 requests/minute                                       │
│                                                                 │
│  1,000 clients:                                                 │
│  └─ 30,000 requests/minute                                      │
│                                                                 │
│  Actual updates in that minute: maybe 5.                        │
│                                                                 │
│  WASTED REQUESTS: 29,995 out of 30,000                          │
│  WASTED BANDWIDTH: 29,995 × (request headers + response body)  │
│  WASTED DATABASE QUERIES: 29,995 version checks                 │
│  WASTED MONEY: every request costs compute                      │
│                                                                 │
│  AND IT'S STILL NOT REAL-TIME.                                  │
│  Worst case latency: 2 seconds (the polling interval).          │
│  Want lower latency? Poll faster. But that means MORE waste.    │
│                                                                 │
│  Polling trades BANDWIDTH for FRESHNESS.                        │
│  You can never have both.                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "There's a fundamental problem here. We're using a tool designed for *asking questions* (HTTP) to solve a problem that requires *being told answers*. We need a different tool."

---

## 1.4 The Phone Call Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE PHONE CALL ANALOGY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP = TEXT MESSAGES 💬                                         │
│  ──────────────────────                                         │
│                                                                 │
│  • Each text is independent (new connection each time)          │
│  • You send a text, wait for a reply                            │
│  • The other person CANNOT text you first                       │
│  • Want updates? Keep texting "any news?" every 5 seconds       │
│  • Annoying for both sides                                      │
│                                                                 │
│    Bob: "Any updates?"                                          │
│    Server: "No."                                                │
│    Bob: "Any updates?"                                          │
│    Server: "No."                                                │
│    Bob: "Any updates?"                                          │
│    Server: "No."                                                │
│    Bob: "Any updates?"                                          │
│    Server: "Yes! Alice finished a task."                        │
│    Bob: "Cool. Any updates?"                                    │
│    Server: "No."  ... 😩                                        │
│                                                                 │
│                                                                 │
│  WEBSOCKET = PHONE CALL 📞                                      │
│  ──────────────────────────                                     │
│                                                                 │
│  • You dial once (handshake)                                    │
│  • The line stays open                                          │
│  • EITHER side can speak at ANY time                            │
│  • When something happens, they just tell you                   │
│  • No wasted "any news?" — silence until there IS news          │
│  • The call ends when someone hangs up                          │
│                                                                 │
│    Bob: *dials server*                                          │
│    Server: "Connected. I'll let you know when things change."   │
│    (10 seconds of silence — and that's FINE)                    │
│    Server: "Alice just completed 'Design API schema'."          │
│    (30 seconds of silence)                                      │
│    Server: "New task added: 'Write documentation'."             │
│    Bob: "Mark task 3 as in_progress."                           │
│    Server: "Done. Notifying other connected users."             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to programming:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Phone Call                  │  WebSocket                       │
│  ────────────────────────────│──────────────────────────────     │
│  Dialing the number          │  Client sends upgrade request    │
│  Ringing... "Hello?"         │  Server calls accept()           │
│  Line is open                │  Persistent TCP connection       │
│  Either person speaks        │  Either side sends messages      │
│  "I have to go, bye"         │  Close frame sent                │
│  Hang up                     │  Connection closed               │
│  Call drops unexpectedly     │  WebSocketDisconnect exception   │
│  Conference call             │  Broadcasting to all clients     │
│  Separate meeting rooms      │  Channels / rooms                │
│  "What's the password        │  Authentication before/after     │
│   to join the call?"         │    accepting connection          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 2: THE PROTOCOL

## 2.1 The Handshake (HTTP → WebSocket Upgrade)

**WebSocket doesn't replace HTTP. It starts as HTTP.**

> "Here's a fact that surprises most people: every WebSocket connection begins its life as a regular HTTP request. The client says, 'Hey, I'd like to upgrade this HTTP connection to a WebSocket.' The server says, 'Sure, let's switch.' After that, the protocol changes entirely."

> "It's like calling a company's front desk (HTTP). You get transferred to a direct line (WebSocket). You needed the front desk to reach the person, but once you're connected, you don't go through the front desk anymore."

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE WEBSOCKET HANDSHAKE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT                                       SERVER            │
│  ──────                                       ──────            │
│     │                                            │              │
│     │  ─── HTTP GET /ws ──────────────────────▶  │              │
│     │      Upgrade: websocket                    │              │
│     │      Connection: Upgrade                   │              │
│     │      Sec-WebSocket-Key: dGhlIH...          │              │
│     │      Sec-WebSocket-Version: 13             │              │
│     │                                            │              │
│     │  This is a NORMAL HTTP request!            │              │
│     │  Your browser or httpx could send this.    │              │
│     │                                            │              │
│     │  ◀── HTTP 101 Switching Protocols ──────── │              │
│     │      Upgrade: websocket                    │              │
│     │      Connection: Upgrade                   │              │
│     │      Sec-WebSocket-Accept: s3pP...         │              │
│     │                                            │              │
│     │  ═══════════════════════════════════════    │              │
│     │  From this point on, it's NO LONGER HTTP.  │              │
│     │  The TCP connection stays open.             │              │
│     │  Both sides send WebSocket FRAMES.          │              │
│     │  ═══════════════════════════════════════    │              │
│     │                                            │              │
│     │  ◀════ message ════▶                       │              │
│     │  ◀════ message ════▶                       │              │
│     │  ◀════ message ════▶                       │              │
│     │                                            │              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key insight:**

> "Notice `HTTP 101 Switching Protocols`. You know your HTTP status codes from Week 3 — 2xx is success, 4xx is client error, 5xx is server error. 101 is in the 1xx 'informational' family. It literally means: 'I acknowledge your request, and we're switching to a different protocol now.' After 101, HTTP is gone. The same TCP connection is now carrying WebSocket frames."

**Why this matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  BECAUSE THE HANDSHAKE IS HTTP:                                 │
│                                                                 │
│  ✅ WebSocket works on port 80/443 (same as HTTP/HTTPS)         │
│  ✅ It passes through most firewalls and proxies                 │
│  ✅ The initial request can carry cookies, query params, headers │
│  ✅ Your existing HTTP infrastructure (nginx, load balancers)    │
│     can route WebSocket connections                             │
│  ✅ FastAPI handles both HTTP and WebSocket on the same app      │
│                                                                 │
│  ws://  = WebSocket over TCP (like http://)                     │
│  wss:// = WebSocket over TLS (like https://)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 The WebSocket Lifecycle

**A WebSocket connection moves through distinct states:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 WEBSOCKET LIFECYCLE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌────────────┐    Handshake     ┌──────────┐                  │
│   │ CONNECTING │ ──────────────▶  │   OPEN   │                  │
│   │            │    (HTTP 101)    │          │                  │
│   └────────────┘                  └─────┬────┘                  │
│                                         │                       │
│                          ┌──────────────┤                       │
│                          │              │                       │
│                          ▼              ▼                       │
│                   ┌────────────┐  ┌────────────┐                │
│                   │  MESSAGES  │  │  CLOSING   │                │
│                   │ ◀═══════▶  │  │ (close     │                │
│                   │  (bi-dir)  │  │  frame     │                │
│                   └──────┬─────┘  │  sent)     │                │
│                          │        └─────┬──────┘                │
│                          │              │                       │
│                          │              ▼                       │
│                          │        ┌────────────┐                │
│                          └──────▶ │   CLOSED   │                │
│                                   │            │                │
│                                   └────────────┘                │
│                                                                 │
│  EVENTS AT EACH STAGE:                                          │
│  ─────────────────────                                          │
│  CONNECTING → Client sends upgrade request                      │
│  OPEN       → Server accepted. Messages can flow.               │
│  MESSAGES   → Either side sends text or binary frames           │
│  CLOSING    → Either side sends a close frame with a code       │
│  CLOSED     → TCP connection torn down. It's over.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How connections end — close codes:**

> "Remember HTTP status codes from Week 3? WebSocket has its own version: close codes. They tell the other side *why* the connection ended."

```
┌─────────────────────────────────────────────────────────────────┐
│                   WEBSOCKET CLOSE CODES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CODE   │ NAME                 │ MEANING                        │
│  ───────│──────────────────────│────────────────────────────     │
│  1000   │ Normal Closure       │ "Goodbye." Clean disconnect.   │
│  1001   │ Going Away           │ Browser tab closed, server     │
│         │                      │ shutting down.                 │
│  1006   │ Abnormal Closure     │ Connection lost. No close      │
│         │                      │ frame received. (network drop) │
│  1008   │ Policy Violation     │ Auth failed, bad behavior.     │
│         │                      │ YOU WILL USE THIS ONE.         │
│  1011   │ Unexpected Condition │ Server hit an error.           │
│         │                      │ Like HTTP 500, but for WS.     │
│                                                                 │
│  ANALOGY:                                                       │
│  1000 = "Thanks for the call, goodbye!"                         │
│  1001 = "Sorry, I have to run — another meeting"                │
│  1006 = Call dropped — tunnel, dead battery                     │
│  1008 = "You're not authorized to be on this call"              │
│  1011 = "Something went wrong on my end, sorry"                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Messages, Not Requests

**This is a fundamental mental shift.**

```
┌─────────────────────────────────────────────────────────────────┐
│            HTTP THINKING vs WEBSOCKET THINKING                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP (what you know):                                          │
│  ─────────────────────                                          │
│  • Every exchange is REQUEST → RESPONSE                         │
│  • 1:1 mapping. Ask a question, get an answer.                  │
│  • Each request has: method, URL, headers, body                 │
│  • Each response has: status code, headers, body                │
│  • Connection opens, exchange happens, connection closes         │
│                                                                 │
│                                                                 │
│  WEBSOCKET (what you're learning):                              │
│  ─────────────────────────────────                              │
│  • Exchanges are MESSAGES. No request/response pairing.         │
│  • Server can send 5 messages without client asking.            │
│  • Client can send 3 messages without expecting replies.        │
│  • No methods, no URLs, no status codes per message.            │
│  • Just raw data: text strings or binary bytes.                 │
│  • Connection opens ONCE, messages flow FREELY, then closes.    │
│                                                                 │
│                                                                 │
│  HTTP:        ──▶ req  ◀── res  ──▶ req  ◀── res               │
│               (paired, rigid, short-lived)                      │
│                                                                 │
│  WebSocket:   ◀══▶ msg  ◀══▶ msg  ◀══▶ msg  ◀══▶ msg           │
│               (unpaired, free-flowing, long-lived)              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Two types of frames:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WEBSOCKET FRAMES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TEXT FRAME                                                     │
│  ──────────                                                     │
│  UTF-8 encoded string. This is what you'll use 99% of the time.│
│  Usually JSON.                                                  │
│                                                                 │
│    '{"type": "task_update", "task_id": 42, "status": "done"}'  │
│                                                                 │
│                                                                 │
│  BINARY FRAME                                                   │
│  ────────────                                                   │
│  Raw bytes. Used for images, audio, files.                      │
│  Rare in typical backend APIs.                                  │
│                                                                 │
│    b'\x89\x50\x4e\x47...'  (PNG image data)                    │
│                                                                 │
│                                                                 │
│  For this course: TEXT FRAMES with JSON payloads.               │
│  It's the universal standard for WebSocket APIs.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 When to Use: WebSocket vs HTTP vs SSE

**Not everything needs a WebSocket.**

```
┌─────────────────────────────────────────────────────────────────┐
│              CHOOSING YOUR COMMUNICATION TOOL                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┬──────────────┬──────────────┬────────────────┐ │
│  │             │    HTTP      │     SSE      │   WebSocket    │ │
│  ├─────────────┼──────────────┼──────────────┼────────────────┤ │
│  │ Direction   │ Client → Srv │ Srv → Client │ Both ◀══▶ Both│ │
│  │             │ (req → res)  │ (push only)  │ (full duplex)  │ │
│  ├─────────────┼──────────────┼──────────────┼────────────────┤ │
│  │ Connection  │ Short-lived  │ Long-lived   │ Long-lived     │ │
│  │             │ (per request)│ (persistent) │ (persistent)   │ │
│  ├─────────────┼──────────────┼──────────────┼────────────────┤ │
│  │ Protocol    │ HTTP         │ HTTP         │ WebSocket (WS) │ │
│  ├─────────────┼──────────────┼──────────────┼────────────────┤ │
│  │ Complexity  │ Low          │ Low          │ Medium         │ │
│  ├─────────────┼──────────────┼──────────────┼────────────────┤ │
│  │ Use when    │ CRUD ops,    │ Live feeds,  │ Chat, collab   │ │
│  │             │ form submits,│ notifications│ editing, games,│ │
│  │             │ data fetches │ dashboards   │ any 2-way      │ │
│  │             │              │ (server push │ real-time       │ │
│  │             │              │  ONLY)       │ interaction     │ │
│  └─────────────┴──────────────┴──────────────┴────────────────┘ │
│                                                                 │
│  THE DECISION:                                                  │
│                                                                 │
│  "Does the SERVER need to push data to the CLIENT?"             │
│      NO  → Use HTTP. You're done.                               │
│      YES → "Does the CLIENT also need to send data BACK         │
│             over the same persistent connection?"               │
│               NO  → SSE is simpler. Use that.                   │
│               YES → WebSocket.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**For your Task Manager project:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  TASK MANAGER: WHAT GOES WHERE?                                 │
│                                                                 │
│  HTTP (keep as-is):                                             │
│  ├─ POST /tasks          — Create a task                        │
│  ├─ GET /tasks           — Fetch task list                      │
│  ├─ PATCH /tasks/{id}    — Update a task                        │
│  └─ DELETE /tasks/{id}   — Delete a task                        │
│                                                                 │
│  WebSocket (new):                                               │
│  ├─ "Task #42 was marked as done"    — push to board viewers    │
│  ├─ "Alice is typing a comment..."   — real-time indicator      │
│  ├─ "New task assigned to you"       — push notification        │
│  └─ Bob sends "I'm viewing project 7" — join a room             │
│                                                                 │
│  HTTP handles the ACTIONS.                                      │
│  WebSocket handles the REACTIONS.                               │
│  They work TOGETHER, not as replacements.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "You do NOT rewrite your entire CRUD API as WebSocket messages. Your REST endpoints stay. WebSocket is an *additional* channel for pushing real-time updates. The PATCH /tasks/{id} endpoint updates the database. Then it *also* pushes a notification over WebSocket to everyone watching that board."

---

# PART 3: FASTAPI WEBSOCKET ENDPOINTS

## 3.1 @app.websocket — Your First WebSocket Route

**If you can write `@app.get`, you can write `@app.websocket`.**

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()


# You already know this:
@app.get("/hello")
async def hello():
    return {"message": "Hello via HTTP"}


# This is new — but the shape is familiar:
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    await websocket.send_text("Hello via WebSocket!")
    await websocket.close()
```

**Side by side:**

```
┌─────────────────────────────────────────────────────────────────┐
│             HTTP ROUTE  vs  WEBSOCKET ROUTE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @app.get("/hello")              @app.websocket("/ws")          │
│  async def hello():              async def ws(websocket):       │
│      return {"msg": "hi"}            await websocket.accept()   │
│                                      await websocket.send_text( │
│                                          "hi"                   │
│                                      )                          │
│                                      await websocket.close()    │
│                                                                 │
│  • Receives Request object       • Receives WebSocket object    │
│  • Returns data (auto-serialized)│  • Must manually accept()    │
│  • One response, then done       │  • Can send many messages    │
│  • FastAPI handles lifecycle     │  • YOU manage the lifecycle   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key difference: YOU control the lifecycle.**

> "With HTTP routes, FastAPI does everything for you. Request comes in, your function runs, returns data, response goes out. Done. With WebSocket, FastAPI gives you the raw connection object and says: 'Here, you manage it.' You decide when to accept, what to send, when to listen, and when to close. More power, more responsibility."

---

## 3.2 The WebSocket Object

**Four core operations — all async:**

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    
    # 1. ACCEPT — "Pick up the phone"
    #    Must be called first. Completes the handshake.
    await websocket.accept()
    
    # 2. SEND — "Speak"
    #    Push data to the client.
    await websocket.send_text("Welcome!")              # Send string
    await websocket.send_json({"type": "greeting"})    # Send dict as JSON
    await websocket.send_bytes(b"\x00\x01")            # Send binary
    
    # 3. RECEIVE — "Listen"
    #    Wait for data from the client.
    text = await websocket.receive_text()              # Get string
    data = await websocket.receive_json()              # Get parsed JSON
    raw = await websocket.receive_bytes()              # Get binary
    
    # 4. CLOSE — "Hang up"
    #    End the connection with a close code.
    await websocket.close(code=1000)                   # Normal closure
```

**Every operation is `async`.** This is where Week 1 pays off:

```
┌─────────────────────────────────────────────────────────────────┐
│                 WHY EVERY OPERATION IS ASYNC                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  await websocket.accept()                                       │
│  └─ Sends the 101 response over the network. Network I/O.      │
│                                                                 │
│  await websocket.send_text("hi")                                │
│  └─ Writes bytes to the TCP socket. Network I/O.                │
│                                                                 │
│  await websocket.receive_text()                                 │
│  └─ Waits for bytes from the TCP socket. Network I/O.           │
│     Could wait SECONDS, MINUTES, or HOURS.                      │
│                                                                 │
│  await websocket.close()                                        │
│  └─ Sends the close frame. Network I/O.                         │
│                                                                 │
│  Remember Week 1: "await" means "I'll wait here — event loop,  │
│  go serve other connections while I wait."                       │
│                                                                 │
│  This is WHY your server can handle thousands of WebSocket      │
│  connections on a single thread. Each connection spends most    │
│  of its time at an `await`, consuming ZERO CPU.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 The Receive Loop (The Pattern That Changes Everything)

**This is the single most important pattern in this lecture.**

> "A WebSocket connection is long-lived. It could stay open for minutes or hours. You need to keep listening for messages. That means a loop."

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    while True:
        # Wait for the next message from the client
        data = await websocket.receive_text()
        
        # Process it, send a response
        await websocket.send_text(f"You said: {data}")
```

**"Wait — `while True`? Isn't that an infinite loop that will burn my CPU?"**

> "If you wrote `while True` in synchronous Python with no sleep, yes — it would burn 100% CPU and freeze everything. But look at what's INSIDE the loop."

```
┌─────────────────────────────────────────────────────────────────┐
│           WHY while True + await IS EFFICIENT                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  while True:                                                    │
│      data = await websocket.receive_text()  ← THE KEY LINE     │
│                     │                                           │
│                     │                                           │
│                     ▼                                           │
│              This YIELDS to the event loop.                     │
│              The coroutine is SUSPENDED.                        │
│              It consumes ZERO CPU.                              │
│              It sits dormant until a message arrives             │
│              on this specific TCP connection.                   │
│                                                                 │
│                                                                 │
│  WITHOUT AWAIT (hypothetical — never do this):                  │
│  ─────────────────────────────────────────────                  │
│      while True:                                                │
│          data = websocket.receive()   # BLOCKING                │
│                                                                 │
│      CPU: 🔥🔥🔥🔥🔥 100%, event loop frozen, all              │
│      other connections dead                                     │
│                                                                 │
│                                                                 │
│  WITH AWAIT (correct):                                          │
│  ─────────────────────                                          │
│      while True:                                                │
│          data = await websocket.receive_text()  # YIELDING      │
│                                                                 │
│      CPU: 😴 sleeping. Event loop free.                         │
│      10,000 connections can all sit at this `await`,            │
│      each consuming near-zero resources.                        │
│                                                                 │
│                                                                 │
│  THIS IS THE PAYOFF OF WEEK 1.                                  │
│  This is WHY you learned async.                                 │
│  A single server thread handles thousands of persistent         │
│  connections because every one of them is suspended at          │
│  an `await`, not burning CPU.                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visualize 3 simultaneous connections:**

```
┌─────────────────────────────────────────────────────────────────┐
│         THREE CLIENTS, ONE SERVER THREAD                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Time ───────────────────────────────────────────────▶          │
│                                                                 │
│  Client A:  [accept] [await···] [recv→send] [await·········]   │
│  Client B:  [accept] [await·········] [recv→send] [await···]   │
│  Client C:  [accept] [await···············] [recv→send] [aw·]  │
│                                                                 │
│  Event Loop: ──┬──┬──┬─────────┬──┬────────┬──┬────────┬──     │
│                │  │  │         │  │        │  │        │        │
│                ▼  ▼  ▼         ▼  ▼        ▼  ▼        ▼        │
│               A  B  C     A msg  B msg   C msg                  │
│             accept    (event loop wakes    (event loop wakes    │
│              all      the right coroutine   the right coroutine │
│                       when its message      when its message    │
│                       arrives)              arrives)            │
│                                                                 │
│  Each "await···" costs nearly ZERO. The event loop just         │
│  checks: "Any data on these sockets?" If yes, resume that      │
│  coroutine. If no, keep waiting. Same model as Week 1's         │
│  restaurant: the waiter checks each table, not standing         │
│  at any single one.                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Complete echo server — the "hello world" of WebSocket:**

```python
# echo_server.py
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/ws/echo")
async def echo(websocket: WebSocket):
    await websocket.accept()
    print("Client connected")
    
    try:
        while True:
            message = await websocket.receive_text()
            print(f"Received: {message}")
            await websocket.send_text(f"Echo: {message}")
    except WebSocketDisconnect:
        print("Client disconnected")
```

> "Run it with `uvicorn echo_server:app --reload`. That's it. You have a WebSocket server."

---

## 3.4 Handling Disconnections (WebSocketDisconnect)

**Clients disconnect. Always. You must handle it.**

> "In the phone call analogy: people hang up. Sometimes politely ('goodbye'), sometimes abruptly (battery dies). Your server must handle both without crashing."

**When does `WebSocketDisconnect` fire?**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHEN DISCONNECTION HAPPENS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLEAN DISCONNECT:                                              │
│  Client sends close frame → receive_text() raises               │
│  WebSocketDisconnect with code=1000                             │
│  (User clicked "disconnect" or navigated away)                  │
│                                                                 │
│  ABNORMAL DISCONNECT:                                           │
│  Client vanishes → receive_text() eventually raises             │
│  WebSocketDisconnect with code=1006                             │
│  (Network dropped, laptop closed, browser crashed)              │
│                                                                 │
│  In both cases, your while True loop ENDS because               │
│  the exception breaks out of it.                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The standard pattern:**

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Received: {data}")
    except WebSocketDisconnect as exc:
        # exc.code contains the close code (1000, 1001, 1006, etc.)
        print(f"Client disconnected with code: {exc.code}")
        # Clean up: remove from manager, notify others, etc.
```

**Exception handling works exactly as you learned in Week 1:**

```
┌─────────────────────────────────────────────────────────────────┐
│             EXCEPTION FLOW IN WEBSOCKET HANDLER                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  async def websocket_endpoint(websocket):                       │
│      await websocket.accept()                                   │
│      try:                                                       │
│          while True:                                            │
│              data = await websocket.receive_text()               │
│              │                  ▲                                │
│              │  ┌───────────────┘                                │
│              │  │  On next receive, client is gone               │
│              │  │  → raises WebSocketDisconnect                  │
│              │  │  → breaks out of while True                    │
│              │  │  → caught by except                            │
│              ▼  │                                                │
│              process(data)                                       │
│      except WebSocketDisconnect:  ◀── Caught here               │
│          cleanup()                                              │
│                                                                 │
│  Same exception mechanics as Week 1. No new concepts.           │
│  The exception propagates through await, caught by try/except.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What about errors during send?**

```python
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            data = await websocket.receive_text()
            
            # What if processing fails? Handle it like any other code.
            try:
                result = process_message(data)
                await websocket.send_json({"status": "ok", "data": result})
            except ValueError as exc:
                # Don't crash the connection! Send an error message.
                await websocket.send_json({
                    "status": "error",
                    "message": str(exc),
                })
                # The while loop continues — connection stays open.
                
    except WebSocketDisconnect:
        print("Client disconnected")
```

> "Notice the two levels of exception handling. The inner try/except handles *processing errors* — bad input, business logic failures. You send an error message and keep the connection alive. The outer try/except handles *disconnection* — the connection is gone, clean up. Don't mix these up."

---

## 3.5 Structured Messages with JSON

**Raw text strings don't scale. You need structure.**

> "When you built your REST API, you didn't send raw strings — you used Pydantic models with defined schemas. WebSocket messages need the same discipline."

**Define a message protocol:**

```python
# The convention: every message has a "type" and a "payload"
# This is YOUR application protocol on top of WebSocket.

# Client → Server messages:
{"type": "join_room", "payload": {"room_id": "project_42"}}
{"type": "task_update", "payload": {"task_id": 7, "status": "done"}}
{"type": "typing", "payload": {"room_id": "project_42"}}

# Server → Client messages:
{"type": "task_updated", "payload": {"task_id": 7, "status": "done", "by": "alice"}}
{"type": "user_joined", "payload": {"user": "bob", "room_id": "project_42"}}
{"type": "error", "payload": {"message": "Task not found"}}
```

**Using `send_json()` and `receive_json()`:**

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            # Receive a JSON message (parsed to dict automatically)
            data: dict = await websocket.receive_json()
            
            message_type = data.get("type")
            payload = data.get("payload", {})
            
            if message_type == "ping":
                await websocket.send_json({
                    "type": "pong",
                    "payload": {}
                })
            
            elif message_type == "task_update":
                task_id = payload.get("task_id")
                new_status = payload.get("status")
                # Process the update (hit your DB, etc.)
                await websocket.send_json({
                    "type": "task_updated",
                    "payload": {
                        "task_id": task_id,
                        "status": new_status,
                        "confirmed": True,
                    },
                })
            
            else:
                await websocket.send_json({
                    "type": "error",
                    "payload": {
                        "message": f"Unknown message type: {message_type}"
                    },
                })
    
    except WebSocketDisconnect:
        print("Client disconnected")
```

**Optional: validate with Pydantic (connecting to Week 3):**

```python
from pydantic import BaseModel, ValidationError
from typing import Any


class WSMessage(BaseModel):
    type: str
    payload: dict[str, Any] = {}


@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            raw = await websocket.receive_json()
            
            # Validate the message structure
            try:
                message = WSMessage.model_validate(raw)
            except ValidationError as exc:
                await websocket.send_json({
                    "type": "error",
                    "payload": {"message": "Invalid message format"},
                })
                continue  # Don't crash — keep listening
            
            # Now you have a validated, typed message
            await handle_message(websocket, message)
    
    except WebSocketDisconnect:
        print("Client disconnected")
```

> "Same Pydantic skills from Week 3. `model_validate()`, `ValidationError`, field constraints — all reusable. The only difference is you're validating WebSocket messages instead of HTTP request bodies."

---

# PART 4: CONNECTION MANAGEMENT

## 4.1 The Problem: Isolated Handlers

**Each WebSocket handler is an island. That's a problem.**

> "Here's something that might not be obvious. Every time a client connects to `/ws`, FastAPI calls your handler function, creating a new coroutine. That coroutine has its own `websocket` object. It can talk to its OWN client. But it has NO access to any OTHER client's websocket object."

```python
# This handler only knows about ONE client
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            # I can send to THIS client...
            await websocket.send_text(f"Echo: {data}")
            # But how do I send to OTHER connected clients?
            # I don't have their WebSocket objects!
    except WebSocketDisconnect:
        pass
```

**The problem visualized:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  ISOLATED HANDLERS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Alice connects to /ws   → handler_1(websocket_A) is created   │
│  Bob connects to /ws     → handler_2(websocket_B) is created   │
│  Charlie connects to /ws → handler_3(websocket_C) is created   │
│                                                                 │
│  handler_1 has websocket_A. It CANNOT see B or C.              │
│  handler_2 has websocket_B. It CANNOT see A or C.              │
│  handler_3 has websocket_C. It CANNOT see A or B.              │
│                                                                 │
│  Alice sends "Task 5 is done."                                  │
│  handler_1 receives it.                                         │
│  handler_1 needs to tell Bob and Charlie.                       │
│  handler_1 has NO WAY to reach websocket_B or websocket_C.     │
│                                                                 │
│  WE NEED SHARED STATE.                                          │
│  Something that ALL handlers can access.                        │
│  Something that tracks who is connected.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Building a ConnectionManager

**The solution: a shared object that tracks all active connections.**

> "In the phone call analogy, this is the conference call operator. They know who's on the line, they can patch people through, and they can broadcast a message to everyone."

**Start simple — a list and three methods:**

```python
from fastapi import WebSocket


class ConnectionManager:
    """Tracks active WebSocket connections."""
    
    def __init__(self) -> None:
        self.active_connections: list[WebSocket] = []
    
    async def connect(self, websocket: WebSocket) -> None:
        """Accept the connection and start tracking it."""
        await websocket.accept()
        self.active_connections.append(websocket)
    
    def disconnect(self, websocket: WebSocket) -> None:
        """Stop tracking a disconnected client."""
        self.active_connections.remove(websocket)
    
    async def send_personal(
        self, message: str, websocket: WebSocket
    ) -> None:
        """Send a message to one specific client."""
        await websocket.send_text(message)
    
    async def broadcast(self, message: str) -> None:
        """Send a message to ALL connected clients."""
        for connection in self.active_connections:
            await connection.send_text(message)
```

**Wiring it into your FastAPI app:**

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

# ONE instance, shared across all handlers
manager = ConnectionManager()


@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    # Manager accepts AND tracks the connection
    await manager.connect(websocket)
    
    try:
        while True:
            data = await websocket.receive_text()
            # Broadcast to EVERYONE, not just this client
            await manager.broadcast(f"Someone said: {data}")
    except WebSocketDisconnect:
        # Remove from tracking, notify others
        manager.disconnect(websocket)
        await manager.broadcast("A user has left.")
```

**Why this works:**

```
┌─────────────────────────────────────────────────────────────────┐
│              CONNECTION MANAGER ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                  ┌───────────────────────┐                      │
│                  │  ConnectionManager    │                      │
│                  │                       │                      │
│                  │  active_connections:  │                      │
│                  │  ├─ websocket_A       │                      │
│                  │  ├─ websocket_B       │                      │
│                  │  └─ websocket_C       │                      │
│                  └──────────┬────────────┘                      │
│                     ▲       │       ▲                           │
│                     │       │       │                           │
│              ┌──────┘       │       └──────┐                    │
│              │              │              │                    │
│        ┌─────┴─────┐ ┌─────┴─────┐ ┌─────┴─────┐              │
│        │ handler_1 │ │ handler_2 │ │ handler_3 │              │
│        │ (Alice)   │ │ (Bob)     │ │ (Charlie) │              │
│        └───────────┘ └───────────┘ └───────────┘              │
│                                                                 │
│  All three handlers share the SAME manager instance.            │
│  When Alice's handler calls manager.broadcast(),                │
│  the manager iterates over [ws_A, ws_B, ws_C] and sends        │
│  the message to each one.                                       │
│                                                                 │
│  Bob's handler doesn't need to know about Alice or Charlie.     │
│  The manager handles the routing.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Broadcasting to All Clients

**Broadcasting needs to be fault-tolerant.**

> "What happens if Bob's connection silently died (laptop closed, no close frame) and the manager tries to send to his dead socket? It raises an exception. And if you don't handle it, that exception kills the ENTIRE broadcast — Charlie never gets the message either."

**Robust broadcast:**

```python
class ConnectionManager:
    
    def __init__(self) -> None:
        self.active_connections: list[WebSocket] = []
    
    async def connect(self, websocket: WebSocket) -> None:
        await websocket.accept()
        self.active_connections.append(websocket)
    
    def disconnect(self, websocket: WebSocket) -> None:
        self.active_connections.remove(websocket)
    
    async def broadcast(self, message: str) -> None:
        """Send to all. If one fails, the others still get it."""
        dead_connections: list[WebSocket] = []
        
        for connection in self.active_connections:
            try:
                await connection.send_text(message)
            except Exception:
                # This connection is dead — mark for removal
                dead_connections.append(connection)
        
        # Clean up dead connections AFTER iterating
        # (never modify a list while iterating over it)
        for dead in dead_connections:
            self.active_connections.remove(dead)
    
    async def broadcast_json(self, data: dict) -> None:
        """Broadcast structured JSON data."""
        dead_connections: list[WebSocket] = []
        
        for connection in self.active_connections:
            try:
                await connection.send_json(data)
            except Exception:
                dead_connections.append(connection)
        
        for dead in dead_connections:
            self.active_connections.remove(dead)
```

**Integration with your existing REST endpoints:**

> "This is where it clicks. Your HTTP endpoints stay. Your WebSocket adds real-time push."

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from pydantic import BaseModel

app = FastAPI()
manager = ConnectionManager()


class TaskUpdate(BaseModel):
    status: str


# Existing HTTP endpoint — unchanged
@app.patch("/tasks/{task_id}")
async def update_task(task_id: int, update: TaskUpdate):
    # Update the database (your existing code)
    updated_task = await task_repository.update(task_id, update)
    
    # NEW: Push real-time notification to all connected clients
    await manager.broadcast_json({
        "type": "task_updated",
        "payload": {
            "task_id": task_id,
            "status": update.status,
        },
    })
    
    return updated_task


# WebSocket endpoint — listens for client messages
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_json()
            # Handle client-to-server messages here
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

> "See the flow? Alice's browser calls `PATCH /tasks/42`. Your HTTP handler updates the database and then calls `manager.broadcast_json()`. Bob and Charlie, who are connected via WebSocket, immediately receive the update. No polling. No delay. Alice used HTTP, Bob and Charlie received via WebSocket. Two protocols working together."

---

## 4.4 Rooms and Channels (Targeted Broadcasting)

**Broadcasting to EVERYONE is rarely what you want.**

> "In your Task Manager, Bob is viewing Project A. Charlie is viewing Project B. When Alice updates a task in Project A, should Charlie get notified? No. He doesn't care. He should only get notifications about Project B."

> "Think of it like Week 11's Redis pub/sub channels. Clients subscribe to specific channels. Messages go only to subscribers of that channel. Same concept, but in-memory within your process."

**Evolve the ConnectionManager:**

```python
from fastapi import WebSocket


class ConnectionManager:
    """Manages WebSocket connections with room support."""
    
    def __init__(self) -> None:
        # Room name → list of connections in that room
        self.rooms: dict[str, list[WebSocket]] = {}
        # Also track all connections for global broadcasts
        self.active_connections: list[WebSocket] = []
    
    async def connect(
        self, websocket: WebSocket, room: str
    ) -> None:
        """Accept connection and add to a room."""
        await websocket.accept()
        self.active_connections.append(websocket)
        
        if room not in self.rooms:
            self.rooms[room] = []
        self.rooms[room].append(websocket)
    
    def disconnect(
        self, websocket: WebSocket, room: str
    ) -> None:
        """Remove connection from room and global tracking."""
        self.active_connections.remove(websocket)
        self.rooms[room].remove(websocket)
        
        # Clean up empty rooms
        if not self.rooms[room]:
            del self.rooms[room]
    
    async def broadcast_to_room(
        self, data: dict, room: str
    ) -> None:
        """Send message to all connections in a specific room."""
        dead: list[WebSocket] = []
        
        for connection in self.rooms.get(room, []):
            try:
                await connection.send_json(data)
            except Exception:
                dead.append(connection)
        
        for d in dead:
            self.rooms[room].remove(d)
            self.active_connections.remove(d)
    
    async def broadcast_all(self, data: dict) -> None:
        """Send to everyone across all rooms (system announcements)."""
        dead: list[WebSocket] = []
        
        for connection in self.active_connections:
            try:
                await connection.send_json(data)
            except Exception:
                dead.append(connection)
        
        for d in dead:
            self.active_connections.remove(d)
```

**Using rooms with path parameters:**

```python
app = FastAPI()
manager = ConnectionManager()


@app.websocket("/ws/projects/{project_id}")
async def project_feed(websocket: WebSocket, project_id: str):
    """Each project is a room. Clients join by connecting."""
    room = f"project_{project_id}"
    
    await manager.connect(websocket, room=room)
    
    # Let others in the room know
    await manager.broadcast_to_room(
        {"type": "user_joined", "payload": {"room": room}},
        room=room,
    )
    
    try:
        while True:
            data = await websocket.receive_json()
            # Handle messages from this client
            # (e.g., typing indicators, live cursor position)
            await manager.broadcast_to_room(data, room=room)
    except WebSocketDisconnect:
        manager.disconnect(websocket, room=room)
        await manager.broadcast_to_room(
            {"type": "user_left", "payload": {"room": room}},
            room=room,
        )


# HTTP endpoint uses the same manager to push to the right room
@app.patch("/projects/{project_id}/tasks/{task_id}")
async def update_task(project_id: str, task_id: int, update: TaskUpdate):
    updated = await task_repository.update(task_id, update)
    
    # Push ONLY to users viewing this project
    await manager.broadcast_to_room(
        {
            "type": "task_updated",
            "payload": {"task_id": task_id, "status": update.status},
        },
        room=f"project_{project_id}",
    )
    
    return updated
```

**Visualize the rooms:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  ROOM-BASED BROADCASTING                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────┐    ┌──────────────────────┐           │
│  │  Room: project_42    │    │  Room: project_99    │           │
│  │                      │    │                      │           │
│  │  ┌─────┐  ┌─────┐   │    │  ┌─────┐             │           │
│  │  │Alice│  │ Bob │   │    │  │Charlie│            │           │
│  │  └─────┘  └─────┘   │    │  └──────┘             │           │
│  │                      │    │                      │           │
│  └──────────────────────┘    └──────────────────────┘           │
│                                                                 │
│  Alice updates task in Project 42.                              │
│  broadcast_to_room("project_42"):                               │
│    ✅ Alice gets it (she's in the room)                          │
│    ✅ Bob gets it (he's in the room)                             │
│    ❌ Charlie does NOT get it (different room)                   │
│                                                                 │
│  Like Week 11 Redis pub/sub — same concept:                     │
│  Rooms ≈ Channels. connect() ≈ SUBSCRIBE. broadcast ≈ PUBLISH.  │
│  Difference: this is in-memory, single-process.                 │
│  (Scaling to multiple processes → Lecture 2 with Redis.)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: WEBSOCKET AUTHENTICATION

## 5.1 The Auth Challenge

**WebSocket auth is tricky. Here's why.**

> "With your REST API, you send `Authorization: Bearer <token>` in the header of every request. Simple. But WebSocket only has ONE HTTP request — the initial handshake. After that, it's not HTTP anymore. There are no headers. There are no requests. Just messages."

```
┌─────────────────────────────────────────────────────────────────┐
│             THE WEBSOCKET AUTH PROBLEM                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP (what you're used to):                                    │
│  ─────────────────────────────                                  │
│  Every request carries the token:                               │
│                                                                 │
│    GET /tasks              Authorization: Bearer eyJ...         │
│    POST /tasks             Authorization: Bearer eyJ...         │
│    PATCH /tasks/1          Authorization: Bearer eyJ...         │
│                                                                 │
│  Your get_current_user dependency (Week 9) extracts and         │
│  validates the token on EVERY request. Clean.                   │
│                                                                 │
│                                                                 │
│  WEBSOCKET (the challenge):                                     │
│  ──────────────────────────                                     │
│  Only ONE HTTP request (the handshake):                         │
│                                                                 │
│    GET /ws  Upgrade: websocket  ← CAN carry headers here       │
│                                                                 │
│    ...connection established...                                 │
│                                                                 │
│    message: {"type": "update"}  ← NO headers here. Not HTTP.   │
│    message: {"type": "chat"}    ← NO headers here. Not HTTP.   │
│                                                                 │
│  You must authenticate ONCE, at connection time.                │
│  Not per-message.                                               │
│                                                                 │
│                                                                 │
│  "But wait — can't I send the header in the handshake?"         │
│                                                                 │
│  In theory: yes. The handshake IS an HTTP request.              │
│  In practice: browser JavaScript's WebSocket API does NOT       │
│  allow setting custom headers. So Authorization: Bearer         │
│  is not an option from browser clients.                         │
│                                                                 │
│  Two workarounds:                                               │
│  1. Token in query parameter: ws://host/ws?token=eyJ...         │
│  2. Token in first message after connecting                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Strategy 1: Token in Query Parameter

**Authenticate BEFORE accepting. Reject immediately if invalid.**

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Query

app = FastAPI()
manager = ConnectionManager()


async def verify_ws_token(token: str) -> dict:
    """Reuse your Week 9 auth logic."""
    # This is your existing verify_access_token function
    # Returns user data if valid, raises if not
    try:
        payload = decode_access_token(token)  # from Week 9
        user = await user_repository.get(payload["sub"])
        if user is None:
            raise ValueError("User not found")
        return user
    except Exception:
        raise ValueError("Invalid token")


@app.websocket("/ws")
async def websocket_endpoint(
    websocket: WebSocket,
    token: str = Query(...),
):
    # Validate BEFORE accepting the connection
    try:
        user = await verify_ws_token(token)
    except ValueError:
        # Reject: close with policy violation code
        await websocket.close(code=1008)
        return  # Handler exits — connection never fully established
    
    # Token valid — accept the connection
    await manager.connect(websocket)
    
    try:
        while True:
            data = await websocket.receive_json()
            # `user` is available for the entire connection lifetime
            await manager.broadcast_json({
                "type": "message",
                "payload": {
                    "from": user.username,
                    "content": data.get("content"),
                },
            })
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

**How the client connects:**

```python
# Client connects with token in URL:
# ws://localhost:8000/ws?token=eyJhbGciOiJIUzI1NiIs...

# Python client example (for testing):
import asyncio
import websockets

async def test_authenticated():
    token = "eyJhbGciOiJIUzI1NiIs..."
    uri = f"ws://localhost:8000/ws?token={token}"
    
    async with websockets.connect(uri) as ws:
        await ws.send('{"content": "Hello!"}')
        response = await ws.recv()
        print(response)

asyncio.run(test_authenticated())
```

**The flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│          QUERY PARAMETER AUTH FLOW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT                                        SERVER           │
│  ──────                                        ──────           │
│     │                                             │             │
│     │── GET /ws?token=eyJ... ──────────────────▶  │             │
│     │   Upgrade: websocket                        │             │
│     │                                             │             │
│     │                          verify_ws_token("eyJ...")        │
│     │                                  │                        │
│     │                           ┌──────┴──────┐                 │
│     │                           │             │                 │
│     │                        VALID         INVALID              │
│     │                           │             │                 │
│     │  ◀── 101 Switching ──────┘             │                 │
│     │      Protocols                          │                 │
│     │                              close(1008)│                 │
│     │  ◀══ messages ══▶          ◀────────────┘                 │
│     │                            Connection rejected.           │
│     │                            Handler exits.                 │
│                                                                 │
│  The token is validated BEFORE accept().                        │
│  Invalid tokens never establish a connection.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Strategy 2: Token in First Message

**Accept first, authenticate second. Close if invalid.**

> "What if you don't want the token in the URL? URLs get logged — in nginx access logs, browser history, server logs. A JWT in a URL is a security concern. The alternative: accept the connection first, then require the client to send their token as the very first message."

```python
import asyncio
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()
manager = ConnectionManager()


@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    # Accept the connection first — it's "unauthenticated" briefly
    await websocket.accept()
    
    # === AUTHENTICATION PHASE ===
    # Client MUST send a valid token within 10 seconds
    try:
        raw = await asyncio.wait_for(
            websocket.receive_json(),
            timeout=10.0,
        )
    except asyncio.TimeoutError:
        await websocket.send_json({
            "type": "error",
            "payload": {"message": "Authentication timeout"},
        })
        await websocket.close(code=1008)
        return
    
    # Expect: {"type": "authenticate", "payload": {"token": "eyJ..."}}
    if raw.get("type") != "authenticate":
        await websocket.send_json({
            "type": "error",
            "payload": {"message": "First message must be authentication"},
        })
        await websocket.close(code=1008)
        return
    
    token = raw.get("payload", {}).get("token")
    try:
        user = await verify_ws_token(token)
    except ValueError:
        await websocket.send_json({
            "type": "error",
            "payload": {"message": "Invalid token"},
        })
        await websocket.close(code=1008)
        return
    
    # === AUTHENTICATED ===
    await websocket.send_json({
        "type": "auth_success",
        "payload": {"user": user.username},
    })
    
    # Now enter the normal message loop
    try:
        while True:
            data = await websocket.receive_json()
            await manager.broadcast_json({
                "type": "message",
                "payload": {
                    "from": user.username,
                    "content": data.get("content"),
                },
            })
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

**Notice `asyncio.wait_for()`** — the timeout prevents unauthenticated connections from hanging forever. This is the same `asyncio` you learned in Week 1, applied to a real security concern.

**The flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│           FIRST MESSAGE AUTH FLOW                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT                                        SERVER           │
│  ──────                                        ──────           │
│     │                                             │             │
│     │── GET /ws ───────────────────────────────▶  │             │
│     │   Upgrade: websocket                        │             │
│     │                                             │             │
│     │  ◀── 101 Switching Protocols ────────────── │             │
│     │                                             │             │
│     │  Connection is OPEN but NOT AUTHENTICATED   │             │
│     │  Server is waiting. 10s countdown started.  │             │
│     │                                             │             │
│     │── {"type": "authenticate",               ─▶ │             │
│     │    "payload": {"token": "eyJ..."}}          │             │
│     │                                             │             │
│     │                         verify_ws_token()   │             │
│     │                               │             │             │
│     │                        ┌──────┴──────┐      │             │
│     │                     VALID         INVALID   │             │
│     │                        │             │      │             │
│     │  ◀─ auth_success ─────┘             │      │             │
│     │                           ◀─ error ──┘      │             │
│     │  ◀══ normal messages ══▶     close(1008)    │             │
│     │                                             │             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 Choosing a Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│            QUERY PARAM vs FIRST MESSAGE AUTH                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                  │  QUERY PARAMETER         FIRST MESSAGE       │
│  ────────────────│──────────────────────   ──────────────────   │
│  Simplicity      │  ✅ Very simple          ⚠️ More complex     │
│                  │                                              │
│  Security        │  ⚠️ Token in URL,        ✅ Token not in URL  │
│                  │  appears in server       sent over encrypted │
│                  │  logs, proxy logs        WebSocket frame     │
│                  │                                              │
│  Auth timing     │  ✅ BEFORE accept()      ⚠️ AFTER accept()   │
│                  │  Invalid tokens never   Connection is        │
│                  │  establish connection    briefly unauthed     │
│                  │                                              │
│  Browser support │  ✅ Works everywhere     ✅ Works everywhere   │
│                  │                                              │
│  Recommendation  │  Good for internal      Better for           │
│                  │  APIs, quick prototyping production, security │
│                  │                          sensitive apps       │
│                  │                                              │
│  ────────────────────────────────────────────────────────────── │
│                                                                 │
│  FOR YOUR PROJECT: start with query parameter for simplicity.   │
│  It's the most common approach in real-world FastAPI apps.      │
│  Switch to first-message if security requirements demand it.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                 WEBSOCKET QUICK REFERENCE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DEFINE A WEBSOCKET ENDPOINT:                                   │
│      @app.websocket("/ws")                                      │
│      async def handler(websocket: WebSocket):                   │
│          ...                                                    │
│                                                                 │
│  ACCEPT THE CONNECTION:                                         │
│      await websocket.accept()                                   │
│                                                                 │
│  SEND DATA:                                                     │
│      await websocket.send_text("hello")                         │
│      await websocket.send_json({"key": "value"})                │
│                                                                 │
│  RECEIVE DATA:                                                  │
│      text = await websocket.receive_text()                      │
│      data = await websocket.receive_json()                      │
│                                                                 │
│  CLOSE THE CONNECTION:                                          │
│      await websocket.close(code=1000)                           │
│                                                                 │
│  THE STANDARD HANDLER PATTERN:                                  │
│      @app.websocket("/ws")                                      │
│      async def handler(websocket: WebSocket):                   │
│          await manager.connect(websocket)                       │
│          try:                                                   │
│              while True:                                        │
│                  data = await websocket.receive_json()           │
│                  # process data, broadcast, etc.                │
│          except WebSocketDisconnect:                             │
│              manager.disconnect(websocket)                       │
│                                                                 │
│  CLOSE CODES:                                                   │
│      1000 = Normal closure                                      │
│      1001 = Going away                                          │
│      1006 = Abnormal (no close frame)                           │
│      1008 = Policy violation (auth failure)                     │
│      1011 = Server error                                        │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ Forgetting await websocket.accept() first               │
│      ❌ Not wrapping while loop in try/except                   │
│         WebSocketDisconnect                                     │
│      ❌ Not cleaning up dead connections in broadcast            │
│      ❌ Broadcasting to everyone when you should use rooms       │
│      ❌ Not authenticating — anyone can connect to /ws           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  WEBSOCKET = PERSISTENT BIDIRECTIONAL CHANNEL                   │
│                                                                 │
│  HTTP handles ACTIONS (create, read, update, delete).           │
│  WebSocket handles REACTIONS (push updates, live events).       │
│  They are PARTNERS, not replacements.                           │
│                                                                 │
│  ┌───────────┐         ┌───────────────┐         ┌───────────┐ │
│  │  Client   │ ──HTTP──▶  Your Server  │──HTTP──▶│ Database  │ │
│  │  Browser  │ ◀═══WS══▶│             │         │           │ │
│  └───────────┘         └───────────────┘         └───────────┘ │
│                                                                 │
│                                                                 │
│  THE PHONE CALL ANALOGY:                                        │
│  ├─ Handshake = Dialing + "Hello?"                              │
│  ├─ accept() = Picking up the phone                             │
│  ├─ send/receive = Talking and listening                        │
│  ├─ while True + await = Keeping the line open, ears ready      │
│  ├─ close(1000) = "Goodbye" — clean hang-up                    │
│  ├─ WebSocketDisconnect = Call dropped                          │
│  ├─ ConnectionManager = Conference call operator                │
│  ├─ Rooms = Separate meeting rooms                              │
│  └─ Auth = "What's the password to join?"                       │
│                                                                 │
│                                                                 │
│  THE ASYNC PAYOFF:                                              │
│  while True:                                                    │
│      data = await websocket.receive_text()                      │
│                   ▲                                             │
│                   │                                             │
│      This single `await` is WHY one server thread               │
│      handles 10,000 simultaneous WebSocket connections.         │
│      Each connection costs near-zero CPU while waiting.         │
│      Week 1 wasn't academic. It was preparation for this.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Upcoming Lectures

```
┌─────────────────────────────────────────────────────────────────┐
│                     WHERE THIS LEADS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 12, LECTURE 2 — Real-Time Patterns:                       │
│  ├─ Heartbeat/ping-pong (detecting dead connections early)      │
│  ├─ Reconnection strategies (what the client does when the      │
│  │   connection drops)                                          │
│  ├─ SCALING: Your ConnectionManager lives in ONE process.       │
│  │   Two uvicorn workers = two managers = split audience.       │
│  │   Solution: Redis pub/sub (Week 11) as the bridge.           │
│  │   This is the #1 production challenge with WebSocket.        │
│  ├─ Server-Sent Events (SSE) — simpler alternative when you    │
│  │   only need server → client push                             │
│  └─ Testing WebSocket endpoints with pytest                     │
│                                                                 │
│  WEEK 12, LECTURE 3 — Performance:                              │
│  └─ Load testing your WebSocket endpoints with locust           │
│     How many concurrent connections can YOUR server handle?     │
│                                                                 │
│  WEEK 12 PROJECT — Add Real-Time & Optimize:                    │
│  └─ You'll add WebSocket notifications to your Task Manager.    │
│     When a task is updated via HTTP, connected users get a      │
│     real-time push. Rooms per project. Authenticated            │
│     connections. Everything from today.                         │
│                                                                 │
│  WEEK 13-14 CAPSTONE:                                           │
│  └─ Real-time notifications via WebSocket is a core feature     │
│     of your SaaS platform. Task updates, mentions,              │
│     team activity feeds — all real-time.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```