# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PRODUCTION FAILURE FIRST                                       │
│  ────────────────────────                                       │
│  Everything you built in Lecture 1 works perfectly — on your    │
│  laptop, with one server, on a stable network. This lecture     │
│  is about what happens when every one of those breaks.          │
│  We show the failure THEN the pattern that prevents it.         │
│                                                                 │
│  SYSTEM THINKING                                                │
│  ───────────────                                                │
│  Students shift from "single-process on localhost" to           │
│  "distributed system with unreliable networks." This is a      │
│  mental shift, not just new API calls.                          │
│                                                                 │
│  RIGHT TOOL, RIGHT PROBLEM                                      │
│  ────────────────────────                                       │
│  WebSocket, SSE, and polling each have their place.             │
│  The decision framework matters more than any implementation.   │
│                                                                 │
│  SYNTHESIS OF PRIOR KNOWLEDGE                                   │
│  ────────────────────────────                                   │
│  Redis pub/sub (Week 11), async tasks (Week 1),                │
│  auth dependencies (Week 9), testing (Week 2/4) —              │
│  this lecture ties them together under real-time architecture.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                     REAL-TIME PATTERNS                          │
│                     (3-4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE GHOST CONNECTION PROBLEM (45 min)                  │
│  ├─ 1.1 Ghosts in Your Server (Demonstration)                   │
│  ├─ 1.2 Why TCP Doesn't Save You                                │
│  ├─ 1.3 The Ping-Pong Protocol                                  │
│  └─ 1.4 Building a Heartbeat Monitor                            │
│                                                                 │
│  PART 2: SURVIVING DISCONNECTIONS (30 min)                      │
│  ├─ 2.1 Why Connections Drop (The Real World)                    │
│  ├─ 2.2 Reconnection from the Server's Perspective              │
│  ├─ 2.3 Missed Message Recovery                                 │
│  └─ 2.4 Grace Periods and State Cleanup                         │
│                                                                 │
│  PART 3: THE SCALING WALL (50 min)                              │
│  ├─ 3.1 One Server Works, Two Servers Break (Demonstration)     │
│  ├─ 3.2 The Architecture Shift                                  │
│  ├─ 3.3 Redis Pub/Sub as the Backbone (Connection to Week 11)   │
│  └─ 3.4 Implementing Cross-Server Broadcasting                  │
│                                                                 │
│  PART 4: SERVER-SENT EVENTS — THE SIMPLER TOOL (30 min)         │
│  ├─ 4.1 Not Everything Needs WebSocket                          │
│  ├─ 4.2 How SSE Works (Protocol and Format)                     │
│  ├─ 4.3 SSE in FastAPI                                          │
│  └─ 4.4 WebSocket vs SSE vs Polling (Decision Framework)        │
│                                                                 │
│  PART 5: TESTING REAL-TIME ENDPOINTS (30 min)                   │
│  ├─ 5.1 Why Real-Time Testing is Hard                           │
│  ├─ 5.2 Testing WebSocket Endpoints with TestClient             │
│  ├─ 5.3 Testing Heartbeat, Disconnect, and Authentication       │
│  └─ 5.4 Mocking Redis for Cross-Server Tests                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE GHOST CONNECTION PROBLEM

## 1.1 Ghosts in Your Server

**Start with the scenario. Make them see the disaster unfolding.**

> "Your chat application from Lecture 1 is deployed. 200 users connected. You check your connection manager: it says 847 active connections. How?"

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE GHOST SCENARIO                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Monday 9:00 AM — 50 users connect                              │
│  ✅ ConnectionManager.active_connections: 50                    │
│                                                                 │
│  Monday 10:30 AM — 30 more connect                              │
│  ✅ ConnectionManager.active_connections: 80                    │
│                                                                 │
│  Monday 12:00 PM — lunch break                                  │
│  20 users close their laptops (wifi off, no close frame sent)   │
│  15 users' phones switch to cellular (TCP connection gone)      │
│  10 users walk into elevator (signal lost)                      │
│                                                                 │
│  ❌ ConnectionManager.active_connections: still 80              │
│                                                                 │
│  Monday 3:00 PM                                                 │
│  50 new users, 60 more ghosts accumulated                       │
│  ❌ ConnectionManager.active_connections: 190                   │
│  ❌ Real living connections: 95                                 │
│  ❌ Ghost connections eating memory: 95                         │
│                                                                 │
│  Wednesday:                                                     │
│  ❌ active_connections: 847                                     │
│  ❌ Real connections: ~200                                      │
│  ❌ Server: running out of memory, broadcast is slow            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now show what happens when you broadcast to ghosts:**

```python
# This is your ConnectionManager from Lecture 1
# It looks correct. It IS correct — for a perfect network.

class ConnectionManager:
    def __init__(self):
        self.active_connections: list[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)
```

**The problem: `disconnect()` is only called when the client sends a proper close frame.** But what about:

```
┌─────────────────────────────────────────────────────────────────┐
│              WAYS A CONNECTION DIES SILENTLY                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ User closes laptop lid (OS suspends network)                │
│     → No close frame sent. Server doesn't know.                │
│                                                                 │
│  ❌ User walks out of wifi range                                │
│     → TCP packets vanish. No RST, no FIN. Just silence.        │
│                                                                 │
│  ❌ User's phone switches from wifi to cellular                 │
│     → New IP address. Old TCP connection is abandoned.          │
│                                                                 │
│  ❌ Intermediate proxy/NAT times out the connection             │
│     → Proxy drops it after inactivity. Neither side knows.     │
│                                                                 │
│  ❌ Client-side crash or OOM kill                               │
│     → Process died. No cleanup code ran.                        │
│                                                                 │
│  ALL OF THESE leave a GHOST CONNECTION on your server:          │
│  A WebSocket object in your list that is connected to nothing.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What does broadcasting to a ghost look like?**

```python
async def broadcast(self, message: str):
    for connection in self.active_connections:
        await connection.send_text(message)
        #     ^^^^^^^^^^^^^^^^^^^^^^^^^^
        #     For a ghost connection, this either:
        #
        #     1. HANGS for minutes (OS TCP timeout, often 15-30 min)
        #        Your broadcast loop is BLOCKED waiting for one ghost.
        #        All other users wait for their messages.
        #
        #     2. Eventually raises an exception (after the OS gives up)
        #        But by then, you've wasted minutes of wall-clock time.
        #
        #     3. Silently succeeds (data goes into OS send buffer)
        #        The ghost stays, memory leaks, buffer fills up.
```

**Ask the class:**

> "Your `broadcast()` iterates through 847 connections. 647 are ghosts. Some hang for 30 seconds each. How long does a single broadcast take?"

> "And this is a chat application — broadcasts happen constantly. Every message waits behind dead connections. What does the user experience look like?"

---

## 1.2 Why TCP Doesn't Save You

**Students might wonder: "Doesn't TCP detect dead connections?"**

```
┌─────────────────────────────────────────────────────────────────┐
│                 TCP KEEPALIVE — TOO LITTLE, TOO LATE            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TCP has a keepalive mechanism. It's almost useless for us.     │
│                                                                 │
│  DEFAULT TCP KEEPALIVE SETTINGS (Linux):                        │
│  ├─ tcp_keepalive_time:     7200 seconds (2 HOURS!)             │
│  │   "Wait 2 hours of silence before checking"                  │
│  ├─ tcp_keepalive_intvl:    75 seconds                          │
│  │   "Then send a probe every 75 seconds"                       │
│  └─ tcp_keepalive_probes:   9                                   │
│      "Give up after 9 failed probes"                            │
│                                                                 │
│  TOTAL TIME TO DETECT DEAD CONNECTION:                          │
│  7200 + (75 × 9) = 7875 seconds ≈ 2 hours 11 minutes           │
│                                                                 │
│  For a chat app, 2 hours to detect a ghost is useless.          │
│  You need seconds, not hours.                                   │
│                                                                 │
│  CAN YOU TUNE TCP KEEPALIVE?                                    │
│  Yes, but:                                                      │
│  ├─ Requires OS-level socket configuration                      │
│  ├─ Applies globally or per-socket (not per-WebSocket)          │
│  ├─ Doesn't work through some NATs and proxies                  │
│  └─ Gives you no application-level information                  │
│                                                                 │
│  WE NEED AN APPLICATION-LEVEL SOLUTION.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 The Ping-Pong Protocol

**The solution: actively probe connections from your application code.**

Think of it like a phone call during a long silence:

```
┌─────────────────────────────────────────────────────────────────┐
│                THE PHONE CALL ANALOGY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT HEARTBEAT:                                             │
│  ──────────────────                                             │
│                                                                 │
│  You:    "So about the project..."                              │
│  Them:   "Yeah, let me look into it..."                         │
│  ...                                                            │
│  (45 minutes of silence)                                        │
│  ...                                                            │
│  You:    "Hello? Are you still there?"                          │
│  ...                                                            │
│  You:    "Hello???"                                             │
│  ...                                                            │
│  (They hung up 44 minutes ago.)                                 │
│                                                                 │
│                                                                 │
│  WITH HEARTBEAT:                                                │
│  ────────────────                                               │
│                                                                 │
│  You:    "So about the project..."                              │
│  Them:   "Yeah, let me look into it..."                         │
│  (15 seconds of silence)                                        │
│  You:    "Still there?"                   ← PING                │
│  Them:   "Yep, still looking."            ← PONG                │
│  (15 seconds of silence)                                        │
│  You:    "Still there?"                   ← PING                │
│  ...     (no response for 30 seconds)                           │
│  You:    (hangs up, calls next person)    ← CLOSE               │
│                                                                 │
│  Dead connection detected in under 45 seconds. Not 2 hours.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Two levels of ping-pong:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TWO LEVELS OF HEARTBEAT                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LEVEL 1: PROTOCOL-LEVEL (Uvicorn handles it)                   │
│  ─────────────────────────────────────────────                  │
│  The WebSocket protocol (RFC 6455) defines ping/pong FRAMES.    │
│  These are special control frames, not regular messages.        │
│  The client MUST respond automatically — browser handles it.    │
│                                                                 │
│  Configure in uvicorn:                                          │
│    uvicorn app:app --ws-ping-interval 20 --ws-ping-timeout 20   │
│                                                                 │
│  ✅ Simplest option. Zero application code.                     │
│  ✅ Browser clients respond automatically.                      │
│  ❌ No custom logic (can't track latency, can't log).           │
│  ❌ Not all WebSocket clients handle it correctly.              │
│  ❌ You can't react to a timeout in your code.                  │
│                                                                 │
│                                                                 │
│  LEVEL 2: APPLICATION-LEVEL (You handle it)                     │
│  ──────────────────────────────────────────                     │
│  You send a regular JSON message: {"type": "ping"}              │
│  Client responds with: {"type": "pong"}                         │
│  You track timing yourself.                                     │
│                                                                 │
│  ✅ Full control: custom timeouts, latency tracking, logging.   │
│  ✅ Works with any client (just JSON messages).                 │
│  ✅ Can trigger cleanup logic on timeout.                       │
│  ❌ More code to write and maintain.                            │
│  ❌ Client must implement the pong response.                    │
│                                                                 │
│  IN PRODUCTION: Use BOTH. Uvicorn ping/pong as a safety net,    │
│  application-level heartbeat for control and observability.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The heartbeat protocol flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 HEARTBEAT PROTOCOL                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Server                                            Client       │
│    │                                                  │         │
│    │  (10 seconds of silence)                         │         │
│    │                                                  │         │
│    │─── {"type": "ping"} ───────────────────────────▶│         │
│    │                                                  │         │
│    │◀── {"type": "pong"} ────────────────────────────│         │
│    │                                                  │         │
│    │  ✅ last_seen updated. Connection alive.         │         │
│    │                                                  │         │
│    │  (10 seconds of silence)                         │         │
│    │                                                  │         │
│    │─── {"type": "ping"} ───────────────────────────▶│         │
│    │                                                  │ ☠️      │
│    │  (client crashed / lost network)                 │         │
│    │                                                  │         │
│    │  (30 second timeout passes, no pong received)    │         │
│    │                                                  │         │
│    │─── close(reason="heartbeat timeout") ──────────▶│         │
│    │                                                  │         │
│    │  🧹 Clean up: remove from manager, free memory   │         │
│    │                                                  │         │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight:**

> "We don't wait for a specific pong message. ANY message from the client proves it's alive. A pong is just a nudge for quiet clients. If the client is actively sending chat messages, those count too — no need to wait for pong."

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "Is the client alive?"                                         │
│                                                                 │
│  Evidence of life = ANY data received from client:              │
│  ├─ {"type": "pong"}              ← explicit heartbeat reply   │
│  ├─ {"type": "message", ...}       ← regular chat message      │
│  ├─ {"type": "typing"}            ← typing indicator            │
│  └─ Any bytes at all               ← the connection is alive    │
│                                                                 │
│  The ping is just a PROMPT for idle connections.                │
│  Active connections prove themselves naturally.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 Building a Heartbeat Monitor

**Let's build this into the connection manager from Lecture 1.**

First, a connection wrapper that tracks liveness:

```python
# heartbeat.py
import asyncio
import time
import logging
from fastapi import WebSocket

logger = logging.getLogger(__name__)


class HeartbeatConnection:
    """Wraps a WebSocket with heartbeat monitoring."""

    def __init__(
        self,
        websocket: WebSocket,
        *,
        ping_interval: float = 15.0,   # Send ping every 15s
        timeout: float = 45.0,         # Dead if silent for 45s
    ):
        self.websocket = websocket
        self.ping_interval = ping_interval
        self.timeout = timeout
        self.last_seen: float = time.time()
        self._heartbeat_task: asyncio.Task | None = None

    def mark_alive(self) -> None:
        """Call on ANY message received from client."""
        self.last_seen = time.time()

    @property
    def is_alive(self) -> bool:
        return (time.time() - self.last_seen) < self.timeout

    @property
    def latency(self) -> float:
        """Seconds since we last heard from this client."""
        return time.time() - self.last_seen

    async def start(self) -> None:
        """Start the background heartbeat loop."""
        self._heartbeat_task = asyncio.create_task(self._heartbeat_loop())

    async def stop(self) -> None:
        """Cancel the heartbeat loop."""
        if self._heartbeat_task is not None:
            self._heartbeat_task.cancel()
            try:
                await self._heartbeat_task
            except asyncio.CancelledError:
                pass

    async def _heartbeat_loop(self) -> None:
        """Periodically ping the client and check for timeout."""
        try:
            while True:
                await asyncio.sleep(self.ping_interval)

                # ── Check: has client spoken recently? ──
                if not self.is_alive:
                    logger.warning(
                        "Heartbeat timeout for connection "
                        f"(silent for {self.latency:.1f}s)"
                    )
                    await self.websocket.close(
                        code=1000,
                        reason="Heartbeat timeout",
                    )
                    return  # Exit the loop — connection is dead

                # ── Still alive: send a ping ──
                try:
                    await self.websocket.send_json({"type": "ping"})
                except Exception:
                    return  # Connection already broken

        except asyncio.CancelledError:
            pass  # Clean shutdown
```

**Walk through the design decisions:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 DESIGN DECISIONS EXPLAINED                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q: Why ping_interval=15, timeout=45?                           │
│  A: Timeout should be > 2× ping_interval.                       │
│     This gives the client at least 2 chances to respond.        │
│     If they miss one ping (network hiccup), they aren't         │
│     immediately killed. If they miss three, they're dead.       │
│                                                                 │
│       ping    ping    ping                                      │
│        │       │       │                                        │
│     0s ├──15s──├──30s──├──45s  ← timeout here                   │
│        │       │       │       │                                │
│        └── 3 chances to respond before being declared dead      │
│                                                                 │
│                                                                 │
│  Q: Why mark_alive() on ANY message, not just pong?             │
│  A: An active user sending chat messages IS alive.              │
│     Requiring a separate pong from active users is wasteful.    │
│     The ping is only needed to poke IDLE connections.           │
│                                                                 │
│                                                                 │
│  Q: Why a separate asyncio.Task? (Connection to Week 1)         │
│  A: The main receive loop (while True: await ws.receive())      │
│     blocks until a message arrives. We can't also check         │
│     timeouts in that same loop. create_task() lets the          │
│     heartbeat run concurrently alongside the receive loop.      │
│     Same pattern as asyncio.gather — two tasks, one thread.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now integrate it into a WebSocket endpoint:**

```python
# main.py
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

from heartbeat import HeartbeatConnection
from manager import ConnectionManager   # Your room-based manager from Lecture 1

app = FastAPI()
manager = ConnectionManager()


@app.websocket("/ws/{room}")
async def websocket_endpoint(websocket: WebSocket, room: str):
    await websocket.accept()

    # ── Wrap connection with heartbeat monitoring ──
    conn = HeartbeatConnection(websocket, ping_interval=15.0, timeout=45.0)
    manager.add(room, conn)
    await conn.start()

    try:
        while True:
            data = await websocket.receive_json()

            # ── ANY message proves the client is alive ──
            conn.mark_alive()

            # ── Handle heartbeat pong (no further processing) ──
            if data.get("type") == "pong":
                continue

            # ── Handle regular messages ──
            await manager.broadcast(room, data)

    except WebSocketDisconnect:
        pass
    finally:
        # ── Always clean up, no matter how we exit ──
        await conn.stop()
        manager.remove(room, conn)
```

**Visualize the two concurrent tasks in the endpoint:**

```
┌─────────────────────────────────────────────────────────────────┐
│       TWO CONCURRENT TASKS PER CONNECTION                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  When a client connects, two tasks run concurrently:            │
│  (Remember asyncio.create_task from Week 1)                     │
│                                                                 │
│                                                                 │
│  TASK 1: Message Receive Loop (the endpoint function)           │
│  ──────────────────────────────────────────────────             │
│  while True:                                                    │
│      data = await websocket.receive_json()   ← paused here     │
│      conn.mark_alive()                                          │
│      if data["type"] == "pong": continue                        │
│      await manager.broadcast(room, data)                        │
│                                                                 │
│                                                                 │
│  TASK 2: Heartbeat Loop (the background task)                   │
│  ──────────────────────────────────────────                     │
│  while True:                                                    │
│      await asyncio.sleep(15)                 ← paused here      │
│      if not self.is_alive:                                      │
│          await websocket.close()                                │
│          return                                                 │
│      await websocket.send_json({"type": "ping"})                │
│                                                                 │
│                                                                 │
│  These alternate on the event loop:                             │
│                                                                 │
│  Time: ───────────────────────────────────────────▶             │
│  Task1: [await recv]          [recv,broadcast]  [await recv]    │
│  Task2:             [sleep 15s]               [check+ping]      │
│                                                                 │
│  Single thread, cooperative multitasking. Week 1 in action.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Updated ConnectionManager that works with HeartbeatConnection:**

```python
# manager.py
import asyncio
import logging
from heartbeat import HeartbeatConnection

logger = logging.getLogger(__name__)


class ConnectionManager:
    """Room-based connection manager with ghost-safe broadcasting."""

    def __init__(self):
        self._rooms: dict[str, set[HeartbeatConnection]] = {}

    def add(self, room: str, conn: HeartbeatConnection) -> None:
        if room not in self._rooms:
            self._rooms[room] = set()
        self._rooms[room].add(conn)

    def remove(self, room: str, conn: HeartbeatConnection) -> None:
        if room in self._rooms:
            self._rooms[room].discard(conn)
            if not self._rooms[room]:
                del self._rooms[room]

    async def broadcast(self, room: str, message: dict) -> None:
        """Broadcast to all live connections in a room."""
        if room not in self._rooms:
            return

        dead: list[HeartbeatConnection] = []

        for conn in self._rooms[room]:
            try:
                await conn.websocket.send_json(message)
            except Exception:
                dead.append(conn)

        # ── Clean up any ghosts discovered during broadcast ──
        for conn in dead:
            logger.info("Removing ghost connection found during broadcast")
            await conn.stop()
            self.remove(room, conn)

    @property
    def total_connections(self) -> int:
        return sum(len(conns) for conns in self._rooms.values())

    @property
    def room_count(self) -> int:
        return len(self._rooms)
```

**Before moving on, summarize what heartbeat solves:**

```
┌─────────────────────────────────────────────────────────────────┐
│              HEARTBEAT RESULTS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT HEARTBEAT:                                             │
│  ├─ Ghost detection: 0 – 2+ hours (TCP keepalive default)      │
│  ├─ Memory: unbounded growth from ghost connections             │
│  ├─ Broadcast: slows down, may hang on dead sockets             │
│  └─ Connection count: unreliable (ghosts inflate the number)    │
│                                                                 │
│  WITH HEARTBEAT (15s interval, 45s timeout):                    │
│  ├─ Ghost detection: 45 seconds max                             │
│  ├─ Memory: bounded, dead connections cleaned promptly          │
│  ├─ Broadcast: fast, only reaches live connections              │
│  └─ Connection count: accurate, trustworthy for monitoring      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 2: SURVIVING DISCONNECTIONS

## 2.1 Why Connections Drop (The Real World)

**Heartbeat detects ghosts. But disconnections aren't always permanent.**

> "Your user is on a train. They go through a tunnel — connection drops. 10 seconds later, they're out of the tunnel and their client reconnects. In those 10 seconds, 15 messages were sent in their group chat. What do they see when they reconnect?"

> "Nothing. Those 15 messages are gone. Your server broadcast them to the old (now dead) connection object. The new connection starts fresh."

```
┌─────────────────────────────────────────────────────────────────┐
│            DISCONNECTIONS ARE NORMAL, NOT EXCEPTIONAL           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TEMPORARY DISCONNECTIONS (seconds to minutes):                 │
│  ├─ Tunnel, elevator, parking garage (signal loss)              │
│  ├─ Wifi → cellular handoff (new IP, old connection dead)       │
│  ├─ ISP hiccup (brief routing failure)                          │
│  └─ Client-side JS garbage collection stall (rare, but real)    │
│                                                                 │
│  MEDIUM DISCONNECTIONS (minutes to hours):                      │
│  ├─ Laptop lid closed on commute (sleep mode)                   │
│  ├─ Phone screen off for a while (OS kills background sockets)  │
│  └─ Network change (switching between wifi networks)            │
│                                                                 │
│  PERMANENT DISCONNECTIONS:                                      │
│  ├─ User navigates away / closes tab                            │
│  ├─ Client crash                                                │
│  └─ User explicitly logs out                                    │
│                                                                 │
│  A ROBUST SYSTEM handles ALL of these gracefully.               │
│  Temporary drops should be INVISIBLE to the user.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 Reconnection from the Server's Perspective

**Reconnection is a CLIENT-SIDE behavior. But the server must SUPPORT it.**

The client's job (you should know this exists, even though you're writing the backend):

```javascript
// CLIENT SIDE (JavaScript, for your awareness)
// A minimal reconnecting WebSocket — this is what your client does.

class ReconnectingSocket {
    constructor(url) {
        this.url = url;
        this.lastMessageId = null;  // Track the last message we saw
        this.attempt = 0;
        this.connect();
    }

    connect() {
        // On reconnect, tell the server what we last received
        const params = this.lastMessageId
            ? `?last_message_id=${this.lastMessageId}`
            : "";
        this.ws = new WebSocket(this.url + params);

        this.ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            if (data.id) this.lastMessageId = data.id;
        };

        this.ws.onclose = () => {
            // Exponential backoff: 1s, 2s, 4s, 8s, ..., max 30s
            const delay = Math.min(1000 * 2 ** this.attempt, 30000);
            this.attempt++;
            setTimeout(() => this.connect(), delay);
        };

        this.ws.onopen = () => {
            this.attempt = 0;  // Reset backoff on successful connect
        };
    }
}
```

**The server's job — what YOU build:**

```
┌─────────────────────────────────────────────────────────────────┐
│          SERVER'S RECONNECTION RESPONSIBILITIES                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ACCEPT last_message_id from reconnecting clients            │
│     "Where did you leave off?"                                  │
│                                                                 │
│  2. REPLAY missed messages since that ID                        │
│     "Here's everything you missed."                             │
│                                                                 │
│  3. ASSIGN every message a sequential ID                        │
│     So clients can track their position.                        │
│                                                                 │
│  4. BUFFER recent messages (for replay on reconnection)         │
│     Can't replay if you didn't store them.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Missed Message Recovery

**Use Redis (Week 10) as a short-term message buffer.**

> "We're not building a permanent message history — that's your PostgreSQL database's job. We're building a SHORT-TERM buffer for reconnection recovery. Messages live in Redis for a few minutes, long enough for a tunnel or elevator ride."

```python
# message_buffer.py
import json
import redis.asyncio as redis      # Week 10 — you know this


class MessageBuffer:
    """
    Short-term message buffer in Redis for reconnection recovery.

    Uses a Redis Sorted Set per room:
      - Score  = message ID (auto-incrementing)
      - Value  = JSON message payload
    Automatically trims old messages to bound memory.
    """

    def __init__(
        self,
        redis_client: redis.Redis,
        max_messages: int = 500,     # Keep last 500 messages per room
        ttl: int = 300,              # Expire after 5 minutes
    ):
        self._redis = redis_client
        self._max_messages = max_messages
        self._ttl = ttl

    async def store(self, room: str, message: dict) -> int:
        """Store a message. Returns the assigned message ID."""
        key = f"buffer:{room}"

        # ── Assign a sequential ID ──
        msg_id = await self._redis.incr(f"buffer:{room}:seq")
        message["id"] = msg_id

        # ── Store in sorted set (score = ID for ordering) ──
        await self._redis.zadd(key, {json.dumps(message): msg_id})

        # ── Trim to max_messages (remove oldest) ──
        await self._redis.zremrangebyrank(key, 0, -(self._max_messages + 1))

        # ── Reset TTL (keep alive while room is active) ──
        await self._redis.expire(key, self._ttl)

        return msg_id

    async def get_since(self, room: str, since_id: int) -> list[dict]:
        """Get all messages with ID > since_id."""
        key = f"buffer:{room}"

        # Sorted set range by score: (since_id, +inf)
        raw_messages = await self._redis.zrangebyscore(
            key,
            min=since_id + 1,    # Exclusive of since_id
            max="+inf",
        )

        return [json.loads(msg) for msg in raw_messages]
```

**Integrate into the WebSocket endpoint:**

```python
@app.websocket("/ws/{room}")
async def websocket_endpoint(
    websocket: WebSocket,
    room: str,
    last_message_id: int | None = None,  # Query param from reconnecting client
):
    await websocket.accept()

    # ── Replay missed messages on reconnection ──
    if last_message_id is not None:
        missed = await message_buffer.get_since(room, last_message_id)
        for msg in missed:
            await websocket.send_json(msg)

    conn = HeartbeatConnection(websocket)
    manager.add(room, conn)
    await conn.start()

    try:
        while True:
            data = await websocket.receive_json()
            conn.mark_alive()

            if data.get("type") == "pong":
                continue

            # ── Store message in buffer BEFORE broadcasting ──
            msg_id = await message_buffer.store(room, data)

            # ── Broadcast (now includes the "id" field) ──
            await manager.broadcast(room, data)

    except WebSocketDisconnect:
        pass
    finally:
        await conn.stop()
        manager.remove(room, conn)
```

**The reconnection flow, end to end:**

```
┌─────────────────────────────────────────────────────────────────┐
│              RECONNECTION FLOW                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FIRST CONNECTION:                                              │
│                                                                 │
│  Client ──── ws://server/ws/room1 ───────────────▶ Server       │
│              (no last_message_id)                                │
│  Client ◀──── messages: id=1, id=2, id=3 ────── Server         │
│                                                                 │
│  Client tracks: lastMessageId = 3                               │
│                                                                 │
│                                                                 │
│  DISCONNECTION (tunnel):                                        │
│                                                                 │
│  Client ─ ─ ─ ╳ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   Server         │
│                                                                 │
│  Meanwhile, server broadcasts id=4, id=5, id=6 to others.      │
│  Messages are stored in Redis buffer.                           │
│                                                                 │
│                                                                 │
│  RECONNECTION:                                                  │
│                                                                 │
│  Client ──── ws://server/ws/room1?last_message_id=3 ──▶ Server  │
│                                                                 │
│  Server: "Since ID 3? Let me check the buffer..."               │
│  Server: buffer.get_since("room1", 3) → [msg4, msg5, msg6]     │
│                                                                 │
│  Client ◀──── replay: id=4, id=5, id=6 ─────── Server          │
│  Client ◀──── live: id=7, id=8, ... ──────────── Server         │
│                                                                 │
│  SEAMLESS. User sees no gap.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Grace Periods and State Cleanup

**Don't immediately destroy user state on disconnect.**

> "When a connection drops, you don't know yet if it's temporary or permanent. Don't erase the user's room memberships, cursor position, or typing status instantly. Give them a grace period to come back."

```python
# graceful_state.py
import asyncio
import time
import logging

logger = logging.getLogger(__name__)


class UserPresence:
    """Track user state with a grace period after disconnect."""

    GRACE_PERIOD: float = 60.0  # 1 minute to reconnect

    def __init__(self):
        # user_id → state
        self._states: dict[int, dict] = {}
        # user_id → cleanup task
        self._cleanup_tasks: dict[int, asyncio.Task] = {}

    async def user_connected(self, user_id: int, rooms: set[str]) -> set[str]:
        """
        Handle a user connecting (or reconnecting).
        Returns rooms to rejoin if this is a reconnection.
        """
        if user_id in self._states:
            # ── RECONNECTION: cancel pending cleanup, restore state ──
            if user_id in self._cleanup_tasks:
                self._cleanup_tasks[user_id].cancel()
                del self._cleanup_tasks[user_id]
                logger.info(f"User {user_id} reconnected within grace period")

            state = self._states[user_id]
            state["status"] = "connected"
            state["disconnected_at"] = None
            return state["rooms"]  # Return rooms they were in

        # ── NEW CONNECTION: create fresh state ──
        self._states[user_id] = {
            "status": "connected",
            "rooms": rooms,
            "disconnected_at": None,
        }
        return set()  # No rooms to rejoin

    async def user_disconnected(self, user_id: int) -> None:
        """Mark user as disconnected. Start the grace period countdown."""
        if user_id not in self._states:
            return

        self._states[user_id]["status"] = "disconnected"
        self._states[user_id]["disconnected_at"] = time.time()

        # ── Start a cleanup timer instead of removing immediately ──
        self._cleanup_tasks[user_id] = asyncio.create_task(
            self._delayed_cleanup(user_id)
        )

    async def _delayed_cleanup(self, user_id: int) -> None:
        """Wait for grace period, then remove state if still disconnected."""
        await asyncio.sleep(self.GRACE_PERIOD)

        if (
            user_id in self._states
            and self._states[user_id]["status"] == "disconnected"
        ):
            del self._states[user_id]
            del self._cleanup_tasks[user_id]
            logger.info(f"User {user_id} grace period expired. State removed.")
```

**Visualize the timeline:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 GRACE PERIOD TIMELINE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CASE 1: User reconnects within grace period                    │
│                                                                 │
│  0s          15s                    45s            60s           │
│  │           │                      │              │            │
│  disconnect  │   reconnect!         │              │            │
│  │           │   │                  │              │            │
│  [──── grace period (cleanup scheduled) ─────────────]          │
│              ▲   ▲                                              │
│              │   └── Cleanup CANCELLED. State restored.         │
│              │       User rejoins rooms seamlessly.             │
│              │                                                  │
│                                                                 │
│  CASE 2: User doesn't come back                                │
│                                                                 │
│  0s                                                60s          │
│  │                                                  │           │
│  disconnect                                     cleanup runs    │
│  │                                                  │           │
│  [──── grace period ────────────────────────────────]           │
│                                                     ▲           │
│                                     State removed. Memory freed.│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: THE SCALING WALL

## 3.1 One Server Works, Two Servers Break

**Everything you've built so far has a hidden assumption: one server.**

> "Your chat app is growing. One server handles 1000 connections. You need to support 10,000. Natural instinct: run more servers behind a load balancer. You spin up 3 instances. What breaks?"

```
┌─────────────────────────────────────────────────────────────────┐
│              THE SINGLE-SERVER ILLUSION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ONE SERVER (everything works):                                 │
│                                                                 │
│  ┌────────────────────────────────────────┐                     │
│  │           SERVER (1 instance)          │                     │
│  │                                        │                     │
│  │  ConnectionManager                     │                     │
│  │  ├─ room "general": [Alice, Bob, Eve]  │                     │
│  │  └─ room "random":  [Bob, Charlie]     │                     │
│  │                                        │                     │
│  │  Alice sends "hello" → broadcast to    │                     │
│  │  all 3 in "general" → ✅ everyone      │                     │
│  │  receives it                           │                     │
│  │                                        │                     │
│  └────────────────────────────────────────┘                     │
│                                                                 │
│  Works perfectly. The ConnectionManager has ALL connections.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now scale to 3 servers:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THREE SERVERS (broadcasting breaks):               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌──────────────┐                              │
│                    │ Load Balancer│                              │
│                    └──────┬───────┘                              │
│               ┌───────────┼───────────┐                         │
│               ▼           ▼           ▼                         │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐       │
│  │   Server 1     │ │   Server 2     │ │   Server 3     │       │
│  │                │ │                │ │                │       │
│  │ ConnectionMgr  │ │ ConnectionMgr  │ │ ConnectionMgr  │       │
│  │ room "general":│ │ room "general":│ │ room "general":│       │
│  │   [Alice]      │ │   [Bob]        │ │   [Eve]        │       │
│  │                │ │                │ │                │       │
│  └────────────────┘ └────────────────┘ └────────────────┘       │
│                                                                 │
│  Alice sends "hello" on Server 1.                               │
│  Server 1's ConnectionManager broadcasts to its room "general". │
│  Server 1's room "general" has: [Alice].                        │
│                                                                 │
│  ❌ Bob (Server 2) never sees it.                               │
│  ❌ Eve (Server 3) never sees it.                               │
│  ❌ Only Alice sees her own message.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "The ConnectionManager is an in-memory Python object. It only knows about connections on ITS process. Server 1 has no idea Server 2 exists. How do you make Server 1's message reach Bob on Server 2?"

---

## 3.2 The Architecture Shift

**The problem is that our ConnectionManager is LOCAL. We need a SHARED communication layer between servers.**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE CORE INSIGHT                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LOCAL ConnectionManager:                                       │
│                                                                 │
│    Server 1:  "Send to room general"                            │
│               → sends to Server 1's connections only            │
│               → other servers don't know                        │
│                                                                 │
│                                                                 │
│  What we NEED:                                                  │
│                                                                 │
│    Server 1:  "Send to room general"                            │
│               → publishes to a SHARED channel                   │
│               → ALL servers receive it                          │
│               → each server sends to ITS local connections      │
│                                                                 │
│                                                                 │
│  The shared channel is a MESSAGE BUS.                           │
│  You already know the perfect tool: Redis Pub/Sub (Week 11).   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│           REDIS PUB/SUB AS MESSAGE BUS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│               ┌──────────────┐                                  │
│               │ Load Balancer│                                  │
│               └──────┬───────┘                                  │
│          ┌───────────┼───────────┐                              │
│          ▼           ▼           ▼                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                         │
│  │ Server 1 │ │ Server 2 │ │ Server 3 │                         │
│  │ [Alice]  │ │ [Bob]    │ │ [Eve]    │                         │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘                         │
│       │            │            │                               │
│       │    subscribe("room:general")                            │
│       │            │            │                               │
│       ▼            ▼            ▼                               │
│  ┌─────────────────────────────────────────┐                    │
│  │              REDIS                       │                    │
│  │                                         │                    │
│  │  Channel: "room:general"                │                    │
│  │  Subscribers: Server 1, Server 2, 3     │                    │
│  │                                         │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                 │
│                                                                 │
│  FLOW when Alice sends a message:                               │
│                                                                 │
│  1. Alice's WS message arrives at Server 1                      │
│  2. Server 1 PUBLISHES to Redis channel "room:general"          │
│  3. Redis delivers to ALL subscribers (Server 1, 2, 3)          │
│  4. Each server broadcasts to ITS LOCAL connections             │
│     ├─ Server 1 → sends to Alice  ✅                            │
│     ├─ Server 2 → sends to Bob    ✅                            │
│     └─ Server 3 → sends to Eve    ✅                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Redis Pub/Sub as the Backbone (Connection to Week 11)

> "In Week 11, you learned Redis Pub/Sub for event-driven architecture — decoupling producers from consumers. This is the exact same pattern. The producer is 'the server that received a WebSocket message.' The consumers are 'all servers that have connections in that room.'"

**Quick recall — Redis Pub/Sub operations:**

```python
import redis.asyncio as redis

r = redis.from_url("redis://localhost:6379")

# Publisher (any server)
await r.publish("room:general", '{"user":"Alice","text":"hello"}')

# Subscriber (all servers)
pubsub = r.pubsub()
await pubsub.subscribe("room:general")

async for message in pubsub.listen():
    if message["type"] == "message":
        channel = message["channel"]   # b"room:general"
        data = message["data"]         # b'{"user":"Alice","text":"hello"}'
```

**That's it. You already know the primitive. Now we wire it into the ConnectionManager.**

---

## 3.4 Implementing Cross-Server Broadcasting

**Replace the local-only ConnectionManager with a Redis-backed version:**

```python
# scalable_manager.py
import asyncio
import json
import logging
from typing import Any

import redis.asyncio as redis
from fastapi import WebSocket

logger = logging.getLogger(__name__)


class ScalableConnectionManager:
    """
    Connection manager that works across multiple server instances.

    Architecture:
      - Each server keeps a LOCAL set of WebSocket connections (in-memory).
      - Broadcasting goes THROUGH Redis Pub/Sub, not directly to sockets.
      - A background listener task receives Redis messages and
        forwards them to local connections.

    This means:
      Server 1 publishes → Redis → Server 1, 2, 3 receive → local broadcast
    """

    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self._redis = redis.from_url(redis_url)
        self._pubsub = self._redis.pubsub()
        self._local_rooms: dict[str, set[WebSocket]] = {}
        self._listener_task: asyncio.Task | None = None

    # ── Lifecycle: call on app startup/shutdown ──

    async def start(self) -> None:
        """Start the Redis listener. Call during FastAPI lifespan."""
        self._listener_task = asyncio.create_task(self._redis_listener())
        logger.info("ScalableConnectionManager started")

    async def stop(self) -> None:
        """Stop listening and close Redis. Call during FastAPI lifespan."""
        if self._listener_task:
            self._listener_task.cancel()
            try:
                await self._listener_task
            except asyncio.CancelledError:
                pass
        await self._pubsub.aclose()
        await self._redis.aclose()
        logger.info("ScalableConnectionManager stopped")

    # ── Connection management (local) ──

    async def connect(self, room: str, websocket: WebSocket) -> None:
        await websocket.accept()

        if room not in self._local_rooms:
            self._local_rooms[room] = set()
            # Subscribe to Redis channel for this room
            await self._pubsub.subscribe(f"room:{room}")
            logger.info(f"Subscribed to Redis channel room:{room}")

        self._local_rooms[room].add(websocket)

    async def disconnect(self, room: str, websocket: WebSocket) -> None:
        if room in self._local_rooms:
            self._local_rooms[room].discard(websocket)

            # Unsubscribe if no local connections left in this room
            if not self._local_rooms[room]:
                del self._local_rooms[room]
                await self._pubsub.unsubscribe(f"room:{room}")
                logger.info(f"Unsubscribed from Redis channel room:{room}")

    # ── Publishing (goes through Redis, not direct to sockets) ──

    async def publish(self, room: str, message: dict[str, Any]) -> None:
        """
        Publish a message to a room.
        This does NOT send directly to WebSockets.
        It publishes to Redis. The listener picks it up.
        """
        await self._redis.publish(f"room:{room}", json.dumps(message))

    # ── Local broadcast (called by listener, not by application code) ──

    async def _broadcast_local(self, room: str, raw_message: str) -> None:
        """Send a message to all LOCAL connections in a room."""
        if room not in self._local_rooms:
            return

        dead: list[WebSocket] = []
        for ws in self._local_rooms[room]:
            try:
                await ws.send_text(raw_message)
            except Exception:
                dead.append(ws)

        for ws in dead:
            self._local_rooms[room].discard(ws)

    # ── Redis listener (the bridge between servers) ──

    async def _redis_listener(self) -> None:
        """
        Background task: listen for messages on subscribed Redis
        channels and forward to local WebSocket connections.

        This is the CORE of cross-server broadcasting.
        """
        try:
            async for message in self._pubsub.listen():
                if message["type"] != "message":
                    continue  # Skip subscribe/unsubscribe confirmations

                channel: str = message["channel"].decode()
                room = channel.removeprefix("room:")
                data: str = message["data"].decode()

                await self._broadcast_local(room, data)

        except asyncio.CancelledError:
            pass  # Clean shutdown
        except Exception as e:
            logger.error(f"Redis listener error: {e}")
            # In production: restart the listener or alert
```

**Wire it into FastAPI using lifespan (you know this from Week 8):**

```python
# main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

from scalable_manager import ScalableConnectionManager

manager = ScalableConnectionManager(redis_url="redis://localhost:6379")


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ── Startup ──
    await manager.start()
    yield
    # ── Shutdown ──
    await manager.stop()


app = FastAPI(lifespan=lifespan)


@app.websocket("/ws/{room}")
async def websocket_endpoint(websocket: WebSocket, room: str):
    await manager.connect(room, websocket)
    try:
        while True:
            data = await websocket.receive_json()

            if data.get("type") == "pong":
                continue

            # ── Publish through Redis, NOT direct broadcast ──
            await manager.publish(room, data)
            # The _redis_listener will pick this up and
            # _broadcast_local to all connections on every server.

    except WebSocketDisconnect:
        pass
    finally:
        await manager.disconnect(room, websocket)
```

**Walk through the flow one more time, in code:**

```
┌─────────────────────────────────────────────────────────────────┐
│              MESSAGE FLOW THROUGH THE SYSTEM                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Alice (Server 1) sends {"text": "hello"}                       │
│                                                                 │
│  ① websocket_endpoint receives it                               │
│     data = await websocket.receive_json()                       │
│                                                                 │
│  ② endpoint calls manager.publish("general", data)              │
│     This does: redis.publish("room:general", json.dumps(data))  │
│     NOTE: It does NOT send to WebSockets directly.              │
│                                                                 │
│  ③ Redis delivers to ALL subscribers of "room:general"          │
│     ├─ Server 1's _redis_listener receives it                   │
│     ├─ Server 2's _redis_listener receives it                   │
│     └─ Server 3's _redis_listener receives it                   │
│                                                                 │
│  ④ Each server's _redis_listener calls _broadcast_local()       │
│     ├─ Server 1 → sends to Alice   ✅                           │
│     ├─ Server 2 → sends to Bob     ✅                           │
│     └─ Server 3 → sends to Eve     ✅                           │
│                                                                 │
│  RESULT: Every user in "general" gets the message,              │
│  regardless of which server they're connected to.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Important: the server that PUBLISHED also RECEIVES its own message through Redis.** This is intentional. It simplifies the code — there's one broadcast path, not two. The trade-off is a tiny bit of extra latency for the sender's own echo, but the architectural simplicity is worth it.

```
┌─────────────────────────────────────────────────────────────────┐
│                SCALING CONSIDERATIONS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Redis Pub/Sub limitations you should know:                     │
│                                                                 │
│  ├─ NO PERSISTENCE: If a server is down when a message is       │
│  │   published, it misses it forever. Pub/Sub is fire-and-      │
│  │   forget. (That's why we have the message buffer in Part 2.) │
│  │                                                              │
│  ├─ SINGLE REDIS: If Redis itself goes down, broadcasting       │
│  │   stops across all servers. Redis is a single point of       │
│  │   failure here. (Redis Sentinel/Cluster for production.)     │
│  │                                                              │
│  └─ SCALING LIMIT: With thousands of rooms and millions of      │
│      messages, a single Redis instance may bottleneck. At that  │
│      scale, look at Redis Cluster or dedicated tools like Kafka │
│      (Week 11 awareness).                                       │
│                                                                 │
│  For most applications (< 100K concurrent connections),         │
│  a single Redis instance handles this trivially.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: SERVER-SENT EVENTS — THE SIMPLER TOOL

## 4.1 Not Everything Needs WebSocket

**Pause. Step back. Consider this scenario from your capstone project (Week 13-14):**

> "A user triggers a CSV export as a background Celery job (Week 11). The UI shows a progress bar. The server pushes progress updates: 10%, 25%, 50%, 100%, done. The user never sends anything back."

> "You reach for WebSocket. But think: the client never sends a message after connecting. It only LISTENS. You've opened a two-way communication channel to send data in ONE direction. That's like renting a walkie-talkie set for two people when one of them only ever listens."

```
┌─────────────────────────────────────────────────────────────────┐
│              PUSH-ONLY USE CASES                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  These scenarios only need SERVER → CLIENT:                     │
│                                                                 │
│  ├─ Progress updates (background job: 10%, 25%, 50%, done)      │
│  ├─ Live notifications (new comment, new follower)              │
│  ├─ Dashboard metrics (live request count, CPU usage)           │
│  ├─ Stock ticker / price feed                                   │
│  ├─ News feed updates                                           │
│  └─ Log streaming (tail -f in the browser)                      │
│                                                                 │
│  For ALL of these, Server-Sent Events (SSE) is simpler,        │
│  lighter, and arguably better than WebSocket.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 How SSE Works (Protocol and Format)

**SSE uses plain HTTP. No upgrade, no new protocol. Just a long-lived GET request with a special content type.**

```
┌─────────────────────────────────────────────────────────────────┐
│              SSE vs WEBSOCKET PROTOCOL                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEBSOCKET:                                                     │
│  ──────────                                                     │
│  Client: GET /ws (with Upgrade: websocket header)               │
│  Server: 101 Switching Protocols                                │
│  → New protocol. Bidirectional frames.                          │
│  → Requires special server support.                             │
│                                                                 │
│                                                                 │
│  SSE (Server-Sent Events):                                      │
│  ──────────────────────────                                     │
│  Client: GET /events (Accept: text/event-stream)                │
│  Server: 200 OK (Content-Type: text/event-stream)               │
│  → Standard HTTP. Server keeps sending data.                    │
│  → Works with any HTTP server. No upgrade.                      │
│                                                                 │
│                                                                 │
│  SSE is just an HTTP response that NEVER ENDS.                  │
│  The server keeps writing to the response body.                 │
│  Each "write" is an event the client receives.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The SSE data format is simple text:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SSE EVENT FORMAT                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Each event is plain text, separated by a blank line (\n\n):    │
│                                                                 │
│  MINIMAL EVENT (data only):                                     │
│  ──────────────────────────                                     │
│                                                                 │
│    data: {"progress": 25, "status": "processing"}\n             │
│    \n                                                           │
│                                                                 │
│                                                                 │
│  FULL EVENT (all optional fields):                              │
│  ─────────────────────────────────                              │
│                                                                 │
│    id: 42\n                    ← Event ID (for reconnection!)   │
│    event: progress_update\n    ← Event type (client can filter) │
│    data: {"progress": 25}\n    ← Payload (the actual data)      │
│    retry: 5000\n               ← Reconnect delay in ms          │
│    \n                          ← Blank line = end of event      │
│                                                                 │
│                                                                 │
│  MULTI-LINE DATA:                                               │
│  ────────────────                                               │
│                                                                 │
│    data: first line\n                                           │
│    data: second line\n                                          │
│    \n                                                           │
│                                                                 │
│    (Client receives: "first line\nsecond line")                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**SSE has built-in reconnection — no client-side code needed:**

```
┌─────────────────────────────────────────────────────────────────┐
│              SSE AUTO-RECONNECTION                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The browser's EventSource API handles reconnection             │
│  AUTOMATICALLY. You don't write reconnection logic.             │
│                                                                 │
│  How it works:                                                  │
│                                                                 │
│  1. Server sends events with "id:" fields                       │
│  2. Connection drops                                            │
│  3. Browser waits (default 3s, or "retry:" value)               │
│  4. Browser reconnects, sends header:                           │
│     Last-Event-ID: 42                                           │
│  5. Server reads that header, replays events since ID 42        │
│                                                                 │
│  Compare to WebSocket:                                          │
│  ├─ WebSocket: YOU write all reconnection logic                 │
│  └─ SSE: The BROWSER does it for you                            │
│                                                                 │
│  This is why SSE is so attractive for push-only scenarios.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 SSE in FastAPI

**FastAPI serves SSE through `StreamingResponse` with an async generator.**

> "Remember async generators? An `async def` with `yield` instead of `return`. FastAPI streams each yielded chunk to the client without closing the connection. Week 1's async knowledge makes this straightforward."

**Example 1: Job progress stream (connects to Week 11 — Celery)**

```python
# sse.py
import asyncio
import json

from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse

from celery_app import get_task_status   # Your Celery setup from Week 11

app = FastAPI()


async def job_progress_stream(
    job_id: str,
    request: Request,
) -> AsyncGenerator[str, None]:
    """
    Yields SSE events tracking a background job's progress.
    Ends when the job completes or fails (or client disconnects).
    """
    event_id = 0

    while True:
        # ── Check if the client disconnected ──
        if await request.is_disconnected():
            break

        # ── Poll the job status (Celery task, Week 11) ──
        status = await get_task_status(job_id)

        event_id += 1
        event = {
            "job_id": job_id,
            "progress": status["progress"],
            "status": status["state"],
        }

        # ── Format as SSE ──
        yield (
            f"id: {event_id}\n"
            f"event: progress\n"
            f"data: {json.dumps(event)}\n"
            f"\n"
        )

        # ── Stop streaming when job is terminal ──
        if status["state"] in ("completed", "failed"):
            break

        await asyncio.sleep(1)


@app.get("/jobs/{job_id}/progress")
async def stream_job_progress(job_id: str, request: Request):
    return StreamingResponse(
        job_progress_stream(job_id, request),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",      # Don't cache the stream
            "X-Accel-Buffering": "no",         # Disable nginx buffering
        },
    )
```

**Example 2: Notification stream with reconnection support**

```python
# notifications_sse.py
from collections.abc import AsyncGenerator

from fastapi import FastAPI, Request, Depends
from fastapi.responses import StreamingResponse

from auth import get_current_user, User   # Week 9
from db import get_notifications_since     # Week 6

app = FastAPI()


async def notification_stream(
    user_id: int,
    request: Request,
    last_event_id: int | None = None,
) -> AsyncGenerator[str, None]:
    """SSE stream of user notifications with reconnection support."""

    # ── Replay missed notifications on reconnection ──
    if last_event_id is not None:
        missed = await get_notifications_since(user_id, last_event_id)
        for notif in missed:
            yield (
                f"id: {notif.id}\n"
                f"event: notification\n"
                f"data: {json.dumps(notif.to_dict())}\n"
                f"\n"
            )

    # ── Stream new notifications as they arrive ──
    while True:
        if await request.is_disconnected():
            break

        # In production: use Redis pub/sub or a queue
        # instead of polling the database
        new = await poll_new_notifications(user_id)

        for notif in new:
            yield (
                f"id: {notif.id}\n"
                f"event: notification\n"
                f"data: {json.dumps(notif.to_dict())}\n"
                f"\n"
            )

        await asyncio.sleep(2)


@app.get("/notifications/stream")
async def notification_sse(
    request: Request,
    current_user: User = Depends(get_current_user),    # Week 9 auth
):
    # ── Browser sends Last-Event-ID header on reconnect ──
    raw_id = request.headers.get("Last-Event-ID")
    last_id = int(raw_id) if raw_id else None

    return StreamingResponse(
        notification_stream(current_user.id, request, last_id),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )
```

**How the browser consumes this (for your awareness):**

```javascript
// Client side — just 5 lines. The browser handles everything.
const source = new EventSource("/notifications/stream", {
    // Auth: SSE uses cookies (not Bearer tokens easily)
    // Or use a polyfill library that supports headers
});

source.addEventListener("notification", (event) => {
    const data = JSON.parse(event.data);
    console.log("New notification:", data);
    // event.lastEventId is tracked automatically
});

source.onerror = () => {
    // Browser auto-reconnects. Sends Last-Event-ID header.
    // You don't need to do anything.
    console.log("Connection lost, browser will reconnect...");
};
```

---

## 4.4 WebSocket vs SSE vs Polling (Decision Framework)

```
┌─────────────────────────────────────────────────────────────────┐
│           CHOOSING THE RIGHT REAL-TIME TOOL                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│              START HERE: "Does the client need                  │
│               to send data AFTER connecting?"                   │
│                          │                                      │
│                    ┌─────┴─────┐                                │
│                   YES          NO                               │
│                    │           │                                │
│                    ▼           ▼                                │
│            ┌────────────┐  "How often do                        │
│            │ WEBSOCKET  │   updates happen?"                    │
│            │            │       │                               │
│            │ Chat       │  ┌────┴────┐                          │
│            │ Gaming     │ Frequent  Rare                        │
│            │ Collab     │ (< 30s)   (minutes+)                  │
│            │ editing    │  │          │                          │
│            └────────────┘  ▼          ▼                          │
│                     ┌──────────┐ ┌──────────┐                   │
│                     │   SSE    │ │ POLLING   │                   │
│                     │          │ │           │                   │
│                     │ Progress │ │ Email     │                   │
│                     │ Notifs   │ │ check     │                   │
│                     │ Feeds    │ │ Sync      │                   │
│                     │ Metrics  │ │ (every    │                   │
│                     └──────────┘ │  5 min)   │                   │
│                                  └──────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Detailed comparison:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  FEATURE            │ WEBSOCKET  │ SSE         │ POLLING        │
│  ───────────────────┼────────────┼─────────────┼───────────     │
│  Direction          │ Bi-direct  │ Server→Cli  │ Client→Svr    │
│  Protocol           │ WS (TCP)   │ HTTP        │ HTTP           │
│  Connection         │ Persistent │ Persistent  │ Repeated       │
│  Auto-reconnect     │ Manual     │ Built-in    │ N/A            │
│  Auth               │ Custom*    │ Cookies/URL │ Headers        │
│  Proxy-friendly     │ Sometimes  │ Yes         │ Yes            │
│  HTTP/2 multiplexed │ No         │ Yes         │ Yes            │
│  Browser support    │ All modern │ All modern  │ All            │
│  Complexity         │ High       │ Low         │ Lowest         │
│  Scaling difficulty │ High       │ Medium      │ Low            │
│                                                                 │
│  *WebSocket auth: token in query param or first message         │
│   (you learned this in Lecture 1)                               │
│                                                                 │
│                                                                 │
│  SSE LIMITATIONS YOU SHOULD KNOW:                               │
│  ├─ Text only (no binary data without base64 encoding)          │
│  ├─ Unidirectional (client can't send through the SSE stream)   │
│  ├─ Max ~6 connections per domain in HTTP/1.1 browsers          │
│  │   (not an issue with HTTP/2)                                 │
│  └─ No built-in Bearer token auth (cookies or query params)     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The rule of thumb:**

> "Start with SSE. Upgrade to WebSocket only when the client needs to send data through the persistent connection. If updates are rare, just poll. Choose the simplest tool that satisfies the requirement."

---

# PART 5: TESTING REAL-TIME ENDPOINTS

## 5.1 Why Real-Time Testing is Hard

```
┌─────────────────────────────────────────────────────────────────┐
│           REAL-TIME TESTING CHALLENGES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Compared to REST endpoints (Week 4), real-time tests are       │
│  harder because:                                                │
│                                                                 │
│  1. STATEFUL: A WebSocket test is a multi-step conversation,    │
│     not a single request→response. Order matters.               │
│                                                                 │
│  2. CONCURRENT: Broadcasting tests need multiple clients        │
│     connected simultaneously.                                   │
│                                                                 │
│  3. TIME-DEPENDENT: Heartbeat tests involve waiting for         │
│     intervals and timeouts.                                     │
│                                                                 │
│  4. BIDIRECTIONAL: You send AND receive, interleaved.           │
│     A REST test asserts one response. A WS test asserts a       │
│     sequence of messages.                                       │
│                                                                 │
│  5. EXTERNAL DEPENDENCIES: Scaling tests need Redis.            │
│     You need to mock or use fakeredis.                          │
│                                                                 │
│  FastAPI's TestClient handles most of this with                 │
│  its websocket_connect() context manager.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Testing WebSocket Endpoints with TestClient

**FastAPI's TestClient provides `websocket_connect()` — your primary tool.**

> "Remember from Week 4: TestClient wraps your app and simulates HTTP. It also simulates WebSocket connections. Same import, new method."

```python
# test_websocket.py
from fastapi.testclient import TestClient

from app.main import app

client = TestClient(app)


# ── TEST: Basic connection and echo ──

def test_websocket_connect_and_send():
    """Client connects, sends a message, receives the broadcast."""
    with client.websocket_connect("/ws/general") as ws:
        ws.send_json({"type": "message", "text": "hello"})
        response = ws.receive_json()

        assert response["text"] == "hello"


# ── TEST: Messages include an ID (for reconnection) ──

def test_messages_have_sequential_ids():
    """Every broadcast message should have an 'id' field."""
    with client.websocket_connect("/ws/general") as ws:
        ws.send_json({"type": "message", "text": "first"})
        msg1 = ws.receive_json()

        ws.send_json({"type": "message", "text": "second"})
        msg2 = ws.receive_json()

        assert "id" in msg1
        assert "id" in msg2
        assert msg2["id"] > msg1["id"]  # Sequential ordering


# ── TEST: Broadcasting reaches multiple clients ──

def test_broadcast_to_room():
    """All clients in the same room receive the broadcast."""
    with (
        client.websocket_connect("/ws/general") as ws1,
        client.websocket_connect("/ws/general") as ws2,
    ):
        ws1.send_json({"type": "message", "text": "hello everyone"})

        # Both clients should receive the broadcast
        msg1 = ws1.receive_json()
        msg2 = ws2.receive_json()

        assert msg1["text"] == "hello everyone"
        assert msg2["text"] == "hello everyone"


# ── TEST: Room isolation ──

def test_rooms_are_isolated():
    """Messages in one room don't leak to another."""
    with (
        client.websocket_connect("/ws/room_a") as ws_a,
        client.websocket_connect("/ws/room_b") as ws_b,
    ):
        # Send in room_a
        ws_a.send_json({"type": "message", "text": "room_a only"})
        msg_a = ws_a.receive_json()
        assert msg_a["text"] == "room_a only"

        # Verify room_b didn't receive it by sending its own message
        ws_b.send_json({"type": "message", "text": "room_b check"})
        msg_b = ws_b.receive_json()

        # room_b's first received message should be its own, not room_a's
        assert msg_b["text"] == "room_b check"
```

---

## 5.3 Testing Heartbeat, Disconnect, and Authentication

```python
# test_heartbeat_and_auth.py
import pytest
from fastapi.testclient import TestClient
from starlette.websockets import WebSocketDisconnect

from app.main import app
from app.auth import create_access_token   # Your JWT helper from Week 9

client = TestClient(app)


# ── TEST: Server sends ping messages ──

def test_server_sends_heartbeat_ping():
    """
    After connecting, the server should send a ping
    within the ping_interval.

    NOTE: This test takes ~ping_interval seconds to run.
    Keep ping_interval short in test config.
    """
    with client.websocket_connect("/ws/general") as ws:
        # Wait for the server's ping (test config: interval=2s)
        msg = ws.receive_json()
        assert msg["type"] == "ping"

        # Respond with pong
        ws.send_json({"type": "pong"})

        # Connection should remain alive — send a real message
        ws.send_json({"type": "message", "text": "still alive"})
        response = ws.receive_json()
        assert response["text"] == "still alive"


# ── TEST: Connection closes on heartbeat timeout ──

def test_heartbeat_timeout_closes_connection():
    """
    If the client never responds to pings, the server
    should close the connection after the timeout period.

    Test config: ping_interval=1s, timeout=3s
    """
    with pytest.raises(Exception):  # Connection will close
        with client.websocket_connect("/ws/general") as ws:
            # Receive ping but DON'T respond with pong
            msg = ws.receive_json()
            assert msg["type"] == "ping"

            # Don't send pong. Wait for timeout.
            # Server should close the connection.
            # The next receive will raise an exception.
            while True:
                ws.receive_json()   # Will raise when server closes


# ── TEST: Unauthenticated clients are rejected ──

def test_websocket_rejects_unauthenticated():
    """Protected WebSocket endpoint rejects connections without a token."""
    with pytest.raises(Exception):
        with client.websocket_connect("/ws/protected/general"):
            pass   # Should never reach this line


# ── TEST: Authenticated clients can connect ──

def test_websocket_accepts_valid_token():
    """Protected endpoint accepts connections with a valid JWT."""
    token = create_access_token(data={"sub": "user@example.com"})

    with client.websocket_connect(
        f"/ws/protected/general?token={token}"
    ) as ws:
        ws.send_json({"type": "message", "text": "authenticated hello"})
        response = ws.receive_json()
        assert response["text"] == "authenticated hello"


# ── TEST: Expired tokens are rejected ──

def test_websocket_rejects_expired_token():
    """Expired JWT should result in connection rejection."""
    token = create_access_token(
        data={"sub": "user@example.com"},
        expires_minutes=-1,   # Already expired
    )

    with pytest.raises(Exception):
        with client.websocket_connect(
            f"/ws/protected/general?token={token}"
        ):
            pass
```

---

## 5.4 Mocking Redis for Cross-Server Tests

> "Your ScalableConnectionManager depends on Redis. In unit tests, you don't want a running Redis instance — that's an integration test concern. Use `fakeredis` or dependency overrides (Week 4) to isolate the behavior."

```python
# test_scalable_manager.py
import pytest
import fakeredis.aioredis       # pip install fakeredis[aioredis]

from app.main import app
from app.dependencies import get_redis
from fastapi.testclient import TestClient


@pytest.fixture
def fake_redis():
    """Create an in-memory fake Redis for testing."""
    return fakeredis.aioredis.FakeRedis()


@pytest.fixture
def test_client(fake_redis):
    """
    TestClient with Redis replaced by fakeredis.
    Uses dependency_overrides — same pattern as Week 4.
    """
    async def override_redis():
        return fake_redis

    app.dependency_overrides[get_redis] = override_redis

    with TestClient(app) as client:
        yield client

    app.dependency_overrides.clear()


def test_cross_room_isolation_with_redis(test_client):
    """Even through Redis pub/sub, rooms should be isolated."""
    with (
        test_client.websocket_connect("/ws/room_x") as ws_x,
        test_client.websocket_connect("/ws/room_y") as ws_y,
    ):
        ws_x.send_json({"type": "message", "text": "room_x msg"})
        msg_x = ws_x.receive_json()
        assert msg_x["text"] == "room_x msg"

        ws_y.send_json({"type": "message", "text": "room_y msg"})
        msg_y = ws_y.receive_json()
        assert msg_y["text"] == "room_y msg"
```

**When to use real Redis (integration tests):**

```python
# test_integration_redis.py
import pytest

REDIS_AVAILABLE = check_redis_connection()  # Helper that pings Redis


@pytest.mark.skipif(not REDIS_AVAILABLE, reason="Redis not running")
@pytest.mark.integration
def test_broadcast_through_real_redis():
    """
    Integration test: verify messages actually flow through Redis.
    Only runs when Redis is available (CI pipeline with Docker).
    """
    # ... test with real Redis connection ...
```

```
┌─────────────────────────────────────────────────────────────────┐
│              TESTING STRATEGY SUMMARY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  UNIT TESTS (fast, no external dependencies):                   │
│  ├─ TestClient + websocket_connect()                            │
│  ├─ fakeredis for Redis-dependent code                          │
│  ├─ dependency_overrides for injected services (Week 4)         │
│  └─ Run on every commit                                         │
│                                                                 │
│  INTEGRATION TESTS (slower, need Docker):                       │
│  ├─ Real Redis (docker-compose up)                              │
│  ├─ Test actual pub/sub message flow                            │
│  ├─ @pytest.mark.integration                                    │
│  └─ Run in CI pipeline (Week 15)                                │
│                                                                 │
│  Same testing pyramid from Week 2:                              │
│  Many unit tests, fewer integration tests.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│               REAL-TIME PATTERNS QUICK REFERENCE                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HEARTBEAT:                                                     │
│      conn = HeartbeatConnection(ws, ping_interval=15, timeout=45│)
│      await conn.start()       # Background ping loop            │
│      conn.mark_alive()        # Call on ANY received message    │
│      await conn.stop()        # Cancel on disconnect            │
│                                                                 │
│  RECONNECTION (server-side):                                    │
│      buffer = MessageBuffer(redis, max_messages=500, ttl=300)   │
│      msg_id = await buffer.store(room, message)                 │
│      missed = await buffer.get_since(room, last_id)             │
│      # Replay missed messages on WebSocket connect              │
│                                                                 │
│  SCALING (Redis Pub/Sub bridge):                                │
│      await manager.publish(room, message)  # → Redis            │
│      # _redis_listener() → _broadcast_local()  (automatic)     │
│      # Each server subscribes to rooms it has connections for   │
│                                                                 │
│  SSE ENDPOINT:                                                  │
│      @app.get("/events")                                        │
│      async def sse(request: Request):                           │
│          return StreamingResponse(                               │
│              generator(request),                                │
│              media_type="text/event-stream",                    │
│          )                                                      │
│                                                                 │
│  SSE EVENT FORMAT:                                              │
│      f"id: {id}\nevent: {type}\ndata: {json}\n\n"              │
│                                                                 │
│  SSE RECONNECTION:                                              │
│      last_id = request.headers.get("Last-Event-ID")             │
│      # Browser sends this automatically on reconnect            │
│                                                                 │
│  TESTING:                                                       │
│      with client.websocket_connect("/ws/room") as ws:           │
│          ws.send_json(data)                                     │
│          response = ws.receive_json()                           │
│          assert response["key"] == "value"                      │
│                                                                 │
│  DECISION:                                                      │
│      Client sends data?  → WebSocket                            │
│      Server push only?   → SSE                                  │
│      Updates rare?       → Polling                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  REAL-TIME IN PRODUCTION = HANDLING UNRELIABILITY               │
│                                                                 │
│  Lecture 1 gave you the happy path:                             │
│    connect → send → receive → disconnect cleanly                │
│                                                                 │
│  This lecture gave you the real world:                           │
│                                                                 │
│  ┌────────────────────┐                                         │
│  │  PROBLEM           │──▶  SOLUTION                            │
│  ├────────────────────┤     ─────────                           │
│  │  Ghost connections │──▶  Heartbeat ping/pong                 │
│  │  Missed messages   │──▶  Message buffer + replay             │
│  │  Abrupt disconnect │──▶  Grace period + reconnect support    │
│  │  Multi-server      │──▶  Redis pub/sub bridge                │
│  │  Overkill          │──▶  SSE for push-only scenarios         │
│  └────────────────────┘                                         │
│                                                                 │
│  Every pattern you learned today is a RESPONSE TO A FAILURE.    │
│  If someone asks "why do we need heartbeat?" the answer is      │
│  not "because it's a best practice." The answer is:             │
│  "because ghost connections will eat your server's memory       │
│   and make broadcasts hang."                                    │
│                                                                 │
│  Know the failure. Then the pattern makes sense.                │
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
│  WEEK 12, LECTURE 3: Performance Measurement & Optimization     │
│  └─ You'll profile your WebSocket server under load.            │
│     How many connections can one instance handle?               │
│     Where does it bottleneck — CPU, memory, or Redis?           │
│                                                                 │
│  WEEK 12, LECTURE 4: Rate Limiting Your API                     │
│  └─ Rate limiting applies to WebSocket connections too.         │
│     How do you prevent one client from flooding a room?         │
│     Connection limits per user, message rate limits.            │
│                                                                 │
│  WEEK 12 PROJECT: Add Real-Time & Optimize                      │
│  └─ You'll add WebSocket notifications to your application,    │
│     with heartbeat, Redis-backed broadcasting, and              │
│     load test the whole thing.                                  │
│     Target: 500 req/min, <200ms p95 latency.                   │
│                                                                 │
│  WEEKS 13-14: Capstone Project                                  │
│  └─ Real-time notifications via WebSocket (task updates,        │
│     mentions). SSE for background job progress.                 │
│     Everything from this lecture, production-grade.             │
│                                                                 │
│  WEEK 15: Docker & Deployment                                   │
│  └─ Containerize the WebSocket server alongside Redis.          │
│     Docker Compose: API + worker + Redis + Postgres.            │
│     Scaling means running multiple containers —                 │
│     that's exactly the multi-server problem we solved today.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```