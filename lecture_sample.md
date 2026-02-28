# ASYNC FUNDAMENTALS
## Week 1, Lecture 3: Coroutines, Event Loop, and When to Use Async

---

## LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM FIRST, SYNTAX LAST                                     │
│  ───────────────────────────                                    │
│  Students must FEEL the problem before learning the solution.   │
│  We'll make them wait. Literally.                               │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Async is abstract. We use a restaurant analogy throughout.     │
│  Every concept maps to something tangible.                      │
│                                                                 │
│  BUILD MENTAL MODEL BEFORE CODE                                 │
│  ─────────────────────────────                                  │
│  The event loop diagram comes before asyncio.run().             │
│  Understand the machine before using it.                        │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Type hints → async functions have typed signatures too         │
│  Error handling → async code raises exceptions the same way     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                     ASYNC FUNDAMENTALS                          │
│                     (3-4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (45 min)                                   │
│  ├─ 1.1 The Waiting Game (Demonstration)                        │
│  ├─ 1.2 What is "Blocking"?                                     │
│  ├─ 1.3 I/O-Bound vs CPU-Bound                                  │
│  └─ 1.4 The Restaurant Analogy                                  │
│                                                                 │
│  PART 2: THE MENTAL MODEL (45 min)                              │
│  ├─ 2.1 Concurrency vs Parallelism                              │
│  ├─ 2.2 The Event Loop (The Restaurant Manager)                 │
│  ├─ 2.3 Coroutines (The Pausable Tasks)                         │
│  └─ 2.4 How Await Works (Yielding Control)                      │
│                                                                 │
│  PART 3: THE SYNTAX (60 min)                                    │
│  ├─ 3.1 async def — Defining Coroutines                         │
│  ├─ 3.2 await — The Magic Keyword                               │
│  ├─ 3.3 asyncio.run() — Starting the Engine                     │
│  ├─ 3.4 Type Hints with Async (Connection to Lecture 1)         │
│  └─ 3.5 Error Handling in Async (Connection to Lecture 2)       │
│                                                                 │
│  PART 4: CONCURRENT PATTERNS (45 min)                           │
│  ├─ 4.1 Sequential vs Concurrent Execution                      │
│  ├─ 4.2 asyncio.gather() — Running Tasks Together               │
│  ├─ 4.3 asyncio.create_task() — Fire and Coordinate             │
│  └─ 4.4 Real-World Pattern: Fetching Multiple URLs              │
│                                                                 │
│  PART 5: WHEN TO USE ASYNC (30 min)                             │
│  ├─ 5.1 The Decision Framework                                  │
│  ├─ 5.2 When Async Helps (I/O-Bound)                            │
│  ├─ 5.3 When Async Hurts (CPU-Bound)                            │
│  └─ 5.4 Common Mistakes and Misconceptions                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Waiting Game

**Start with a demonstration. Make them feel the pain.**

```python
# demo_sync.py — Run this with students watching
import time

def fetch_weather(city: str) -> dict:
    """Simulate fetching weather from an API"""
    print(f"Fetching weather for {city}...")
    time.sleep(2)  # Simulate network delay
    print(f"Got weather for {city}")
    return {"city": city, "temp": 20}

def fetch_news(topic: str) -> dict:
    """Simulate fetching news from an API"""
    print(f"Fetching news about {topic}...")
    time.sleep(2)  # Simulate network delay
    print(f"Got news about {topic}")
    return {"topic": topic, "articles": 5}

def fetch_stock(symbol: str) -> dict:
    """Simulate fetching stock price"""
    print(f"Fetching stock price for {symbol}...")
    time.sleep(2)  # Simulate network delay
    print(f"Got stock price for {symbol}")
    return {"symbol": symbol, "price": 150.0}

def main():
    start = time.time()
    
    weather = fetch_weather("London")
    news = fetch_news("Python")
    stock = fetch_stock("AAPL")
    
    elapsed = time.time() - start
    print(f"\nTotal time: {elapsed:.1f} seconds")
    print(f"Results: {weather}, {news}, {stock}")

main()
```

**Run it. Watch the clock.**

```
Fetching weather for London...
Got weather for London
Fetching news about Python...
Got news about Python
Fetching stock price for AAPL...
Got stock price for AAPL

Total time: 6.0 seconds
```

**Now ask the class:**

> "Each request takes 2 seconds. We made 3 requests. Total: 6 seconds. But here's the question: during those 2 seconds waiting for London's weather... what was your CPU doing?"

Answer: **Nothing. Waiting. Idle.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE TIMELINE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Second:  0   1   2   3   4   5   6                             │
│           │   │   │   │   │   │   │                             │
│  Weather: [===WAITING===]                                       │
│  News:                    [===WAITING===]                       │
│  Stock:                                   [===WAITING===]       │
│                                                                 │
│  Your CPU: 😴  😴  😴  😴  😴  😴                                │
│                                                                 │
│  TOTAL: 6 seconds of wall-clock time                            │
│  ACTUAL CPU WORK: Milliseconds                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The insight:**

> "These three requests are INDEPENDENT. There's no reason news has to wait for weather. The only reason it waits is because our code is SYNCHRONOUS — it executes one line at a time, and `time.sleep(2)` blocks everything."

---

## 1.2 What is "Blocking"?

**Define the core concept:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     BLOCKING                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A "blocking" operation is one that STOPS your program from     │
│  doing anything else until it completes.                        │
│                                                                 │
│  Common blocking operations:                                    │
│  ├─ time.sleep()         — Waiting for time to pass            │
│  ├─ requests.get()       — Waiting for HTTP response           │
│  ├─ file.read()          — Waiting for disk to return data     │
│  ├─ socket.recv()        — Waiting for network data            │
│  └─ input()              — Waiting for user to type            │
│                                                                 │
│  When code blocks, the CPU sits idle, waiting for external     │
│  events (network, disk, user, clock) to complete.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visual metaphor:**

```
BLOCKING CODE:
                                                          
  Your Program          Network                           
  ─────────────         ─────────                         
       │                                                  
       │ ──── Request ────────────────▶                   
       │                              │                   
       │   😴 Blocked                 │ (server thinking)
       │   😴 (can't do               │                   
       │   😴  anything)              │                   
       │                              │                   
       │ ◀──────────── Response ──────│                   
       │                                                  
       ▼ Continue                                         
                                                          
```

---

## 1.3 I/O-Bound vs CPU-Bound

**Two types of "slow" — this distinction is critical:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     TWO TYPES OF SLOW                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  I/O-BOUND                          CPU-BOUND                   │
│  ──────────                         ─────────                   │
│  Waiting for INPUT/OUTPUT           Doing CALCULATIONS          │
│                                                                 │
│  Examples:                          Examples:                   │
│  • Fetching from APIs               • Image processing          │
│  • Reading/writing files            • Video encoding            │
│  • Database queries                 • Scientific computation    │
│  • User input                       • Machine learning training │
│                                                                 │
│  CPU during wait: 😴 Idle           CPU during work: 🔥 100%    │
│                                                                 │
│  ASYNC HELPS HERE ✅                ASYNC DOES NOT HELP ❌      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why async helps I/O-bound but not CPU-bound:**

```
I/O-BOUND: CPU is FREE while waiting
─────────────────────────────────────
  Task A: [WAIT-----------]          
  Task B:      [WAIT-----------]     ← Can start during A's wait
  Task C:           [WAIT-----------]← Can start during B's wait
  
  CPU can juggle multiple waiting tasks!

CPU-BOUND: CPU is BUSY the whole time
─────────────────────────────────────
  Task A: [CALCULATE======]
  Task B:                  [CALCULATE======]  ← Must wait
  Task C:                                     [CALCULATE======]
  
  CPU can only do one calculation at a time (per core).
  Async doesn't magically create more CPUs.
```

---

## 1.4 The Restaurant Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE RESTAURANT ANALOGY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SYNCHRONOUS WAITER (Blocking)                                  │
│  ─────────────────────────────                                  │
│                                                                 │
│  1. Take order from Table 1                                     │
│  2. Walk to kitchen, give order to chef                         │
│  3. STAND THERE and wait for food (10 minutes)  ← Blocked!     │
│  4. Bring food to Table 1                                       │
│  5. NOW go to Table 2...                                        │
│                                                                 │
│  Result: 3 tables = 30 minutes. Customers angry. 😠             │
│                                                                 │
│                                                                 │
│  ASYNCHRONOUS WAITER (Non-blocking)                             │
│  ──────────────────────────────────                             │
│                                                                 │
│  1. Take order from Table 1, give to chef                       │
│  2. While Table 1's food cooks → Take order from Table 2        │
│  3. While Table 2's food cooks → Take order from Table 3        │
│  4. Table 1's food ready → Deliver it                           │
│  5. Table 2's food ready → Deliver it                           │
│  6. Table 3's food ready → Deliver it                           │
│                                                                 │
│  Result: 3 tables ≈ 12 minutes. Everyone happy. 😊              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to programming:**

```
Restaurant              │  Python Async
────────────────────────│─────────────────────────
Waiter                  │  Your program
Tables (customers)      │  Tasks to complete
Kitchen (chef cooking)  │  External I/O (network, disk)
"Food is ready" bell    │  Event (I/O completed)
Taking another order    │  Starting another coroutine
  while food cooks      │    while awaiting I/O
Restaurant manager      │  Event loop
  coordinating waiters  │    coordinating coroutines
```

---

# PART 2: THE MENTAL MODEL

## 2.1 Concurrency vs Parallelism

**A critical distinction that confuses everyone:**

```
┌─────────────────────────────────────────────────────────────────┐
│                CONCURRENCY VS PARALLELISM                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONCURRENCY                                                    │
│  ───────────                                                    │
│  "Dealing with multiple things at once"                         │
│  One worker, switching between tasks                            │
│                                                                 │
│     Task A: [##]    [##]    [##]                                │
│     Task B:     [##]    [##]    [##]                            │
│                ─────────────────────▶ time                      │
│                                                                 │
│  Like a chef chopping vegetables, then stirring soup,           │
│  then back to vegetables. One person, multiple tasks.           │
│                                                                 │
│                                                                 │
│  PARALLELISM                                                    │
│  ───────────                                                    │
│  "Doing multiple things at once"                                │
│  Multiple workers, truly simultaneous                           │
│                                                                 │
│     Task A: [################]  ← Worker 1                      │
│     Task B: [################]  ← Worker 2                      │
│                ─────────────────────▶ time                      │
│                                                                 │
│  Like two chefs, each cooking their own dish.                   │
│  Two people, two tasks, same time.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Python async is CONCURRENT, not PARALLEL:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Python async (asyncio):                                        │
│  ├─ CONCURRENT: Yes ✅                                          │
│  ├─ PARALLEL: No ❌ (single thread)                             │
│  └─ Good for: I/O-bound tasks                                   │
│                                                                 │
│  Python threading:                                              │
│  ├─ CONCURRENT: Yes ✅                                          │
│  ├─ PARALLEL: Limited (GIL)                                     │
│  └─ Good for: I/O-bound tasks, some concurrency                 │
│                                                                 │
│  Python multiprocessing:                                        │
│  ├─ CONCURRENT: Yes ✅                                          │
│  ├─ PARALLEL: Yes ✅ (multiple processes)                       │
│  └─ Good for: CPU-bound tasks                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**For this course:**

> "We focus on async because backend development is OVERWHELMINGLY I/O-bound. You're waiting for databases, APIs, files — not doing heavy calculations. Async is the right tool."

---

## 2.2 The Event Loop (The Restaurant Manager)

**The event loop is the heart of async Python.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      THE EVENT LOOP                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Think of it as a RESTAURANT MANAGER who:                       │
│                                                                 │
│  1. Keeps a list of all pending orders (tasks)                  │
│  2. Checks: "Is any food ready?" (I/O completed?)               │
│  3. If yes → Tells waiter to deliver it (resume coroutine)      │
│  4. If no → Moves to next pending order                         │
│  5. Repeats forever until all orders done                       │
│                                                                 │
│                      ┌──────────────┐                           │
│                      │  EVENT LOOP  │                           │
│                      │  (Manager)   │                           │
│                      └──────┬───────┘                           │
│                             │                                   │
│          ┌──────────────────┼──────────────────┐                │
│          │                  │                  │                │
│          ▼                  ▼                  ▼                │
│    ┌──────────┐      ┌──────────┐      ┌──────────┐             │
│    │  Task 1  │      │  Task 2  │      │  Task 3  │             │
│    │ (waiting │      │ (waiting │      │ (waiting │             │
│    │  for I/O)│      │  for I/O)│      │  for I/O)│             │
│    └──────────┘      └──────────┘      └──────────┘             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The event loop cycle:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   EVENT LOOP CYCLE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌─────────────┐                              │
│           ┌───────▶│ Check tasks │◀───────┐                     │
│           │        └──────┬──────┘        │                     │
│           │               │               │                     │
│           │               ▼               │                     │
│           │        ┌─────────────┐        │                     │
│           │        │ Any task    │        │                     │
│           │        │ ready to    │        │                     │
│           │        │ run?        │        │                     │
│           │        └──────┬──────┘        │                     │
│           │               │               │                     │
│           │      Yes      │      No       │                     │
│           │    ┌──────────┴──────────┐    │                     │
│           │    ▼                     ▼    │                     │
│           │  ┌─────────┐      ┌─────────┐ │                     │
│           │  │ Run it  │      │ Wait for│ │                     │
│           │  │ until it│      │ I/O     │ │                     │
│           │  │ awaits  │      │ events  │ │                     │
│           │  └────┬────┘      └────┬────┘ │                     │
│           │       │                │      │                     │
│           └───────┴────────────────┴──────┘                     │
│                                                                 │
│           (Loop continues until all tasks complete)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Coroutines (The Pausable Tasks)

**A coroutine is a function that can PAUSE and RESUME.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      COROUTINES                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REGULAR FUNCTION:                                              │
│  ─────────────────                                              │
│  Runs from start to finish. Cannot pause.                       │
│                                                                 │
│     def regular():              [═══════════════════]           │
│         step1()                  start           finish         │
│         step2()                  │                  │           │
│         step3()                  └──── no pausing ──┘           │
│                                                                 │
│                                                                 │
│  COROUTINE:                                                     │
│  ──────────                                                     │
│  Can pause in the middle. Another coroutine runs. Then resume.  │
│                                                                 │
│     async def coro():           [═══]    [═══]    [═══]         │
│         await step1()             │  pause │  pause │           │
│         await step2()             │   ▼    │   ▼    │           │
│         await step3()          (other coroutines run here)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight:**

> "A coroutine is like a waiter who can put down their notepad mid-task, help another table, then pick up exactly where they left off. The `await` keyword is saying: 'I'm going to wait here — let someone else work while I wait.'"

---

## 2.4 How Await Works (Yielding Control)

**`await` is the PAUSE button:**

```python
async def fetch_data():
    print("1. Starting fetch")
    
    data = await get_from_api()  # ← PAUSE HERE
    # "I'll wait for the API. Event loop, run other tasks."
    # When API responds, resume right here.
    
    print("2. Got data, processing")
    result = process(data)
    
    await save_to_db(result)  # ← PAUSE AGAIN
    # "I'll wait for the database. Event loop, run other tasks."
    # When DB confirms, resume right here.
    
    print("3. Done!")
    return result
```

**Visualize the pausing:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWAIT = YIELD CONTROL                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Time ──────────────────────────────────────────────────▶       │
│                                                                 │
│  fetch_data():                                                  │
│      print("Starting")  ══╗                                     │
│      await get_api()      ║                                     │
│                           ║ "I'll wait. Someone else go."       │
│                           ╠═════════════════════════════════╗   │
│      print("Got data")                                      ║   │
│      await save_db()    ══════════════════════════════════╗ ║   │
│                                                           ║ ║   │
│      print("Done!")     ══════════════════════════════════╝ ║   │
│                                                             ║   │
│  other_task():                                              ║   │
│      (runs here)        ════════════════════════════════════╝   │
│                                                                 │
│                                                                 │
│  The EVENT LOOP decides who runs during each pause.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What can you await?**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHAT CAN YOU AWAIT?                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ CAN AWAIT (Awaitables):                                     │
│  ├─ Coroutines              await some_async_function()         │
│  ├─ asyncio.sleep()         await asyncio.sleep(1)              │
│  ├─ asyncio.gather()        await asyncio.gather(a(), b())      │
│  ├─ Tasks                   await asyncio.create_task(...)      │
│  └─ Async library calls     await httpx.get(...) (Week 4)       │
│                                                                 │
│  ❌ CANNOT AWAIT (Blocking calls):                              │
│  ├─ time.sleep()            BLOCKS the event loop!              │
│  ├─ requests.get()          Use httpx/aiohttp instead           │
│  ├─ Regular file I/O        Use aiofiles instead                │
│  └─ Regular functions       def func() cannot be awaited        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: THE SYNTAX

## 3.1 async def — Defining Coroutines

**The `async` keyword marks a function as a coroutine:**

```python
# Regular function
def greet(name: str) -> str:
    return f"Hello, {name}"

# Coroutine function
async def greet_async(name: str) -> str:
    return f"Hello, {name}"
```

**What's the difference?**

```python
# Regular function returns the value directly
result = greet("Alice")
print(result)  # "Hello, Alice"

# Coroutine function returns a COROUTINE OBJECT
result = greet_async("Alice")
print(result)  # <coroutine object greet_async at 0x...>

# To get the actual value, you must AWAIT it (inside another async function)
# or RUN it with asyncio.run()
```

**The coroutine object is like an uncooked meal:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Regular function call:    greet("Alice")                       │
│                           │                                     │
│                           ▼                                     │
│                     "Hello, Alice"  ← You get the food          │
│                                                                 │
│                                                                 │
│  Async function call:     greet_async("Alice")                  │
│                           │                                     │
│                           ▼                                     │
│                     <coroutine>     ← You get a meal TICKET     │
│                                                                 │
│                     await <coroutine>                           │
│                           │                                     │
│                           ▼                                     │
│                     "Hello, Alice"  ← NOW you get the food      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 await — The Magic Keyword

**`await` does two things:**

1. **Pauses** the current coroutine
2. **Gets** the result when ready

```python
import asyncio

async def slow_operation() -> str:
    print("Starting slow operation...")
    await asyncio.sleep(2)  # Pause for 2 seconds (non-blocking!)
    print("Slow operation complete!")
    return "Result from slow operation"

async def main():
    print("Before await")
    result = await slow_operation()  # Pause here, get result when ready
    print(f"After await: {result}")

asyncio.run(main())
```

Output:
```
Before await
Starting slow operation...
Slow operation complete!
After await: Result from slow operation
```

**CRITICAL: `await` can ONLY be used inside `async def`:**

```python
# ❌ WRONG: await outside async function
def main():
    result = await slow_operation()  # SyntaxError!

# ✅ CORRECT: await inside async function
async def main():
    result = await slow_operation()  # Works!
```

---

## 3.3 asyncio.run() — Starting the Engine

**`asyncio.run()` is the ignition key:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  REGULAR PYTHON             │  ASYNC PYTHON                     │
│  ──────────────             │  ────────────                     │
│                             │                                   │
│  if __name__ == "__main__": │  if __name__ == "__main__":       │
│      main()                 │      asyncio.run(main())          │
│                             │      ▲                            │
│                             │      │                            │
│                             │      └─ Starts the event loop     │
│                             │         Runs until main() done    │
│                             │         Cleans up                 │
│                             │                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Complete example:**

```python
import asyncio

async def fetch_weather(city: str) -> dict:
    print(f"Fetching weather for {city}...")
    await asyncio.sleep(2)  # Simulate API call
    print(f"Got weather for {city}")
    return {"city": city, "temp": 20}

async def main():
    result = await fetch_weather("London")
    print(f"Result: {result}")

# Entry point — starts the async world
asyncio.run(main())
```

**What `asyncio.run()` does:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    asyncio.run(main())                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Creates a new event loop                                    │
│     └─ "Opens the restaurant"                                   │
│                                                                 │
│  2. Runs the coroutine until complete                           │
│     └─ "Serves all customers until everyone's done"             │
│                                                                 │
│  3. Closes the event loop                                       │
│     └─ "Closes the restaurant for the night"                    │
│                                                                 │
│  4. Returns the result of the coroutine                         │
│     └─ "Reports how the night went"                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Rules:**

```python
# ✅ Call asyncio.run() ONCE at the top level
asyncio.run(main())

# ❌ DON'T call asyncio.run() inside an async function
async def bad():
    asyncio.run(other())  # RuntimeError: already running!

# ❌ DON'T call asyncio.run() multiple times (usually)
asyncio.run(task1())
asyncio.run(task2())  # Creates new event loop each time — wasteful
```

---

## 3.4 Type Hints with Async (Connection to Lecture 1)

**Connection to what you've learned:**

> "Remember type hints from Lecture 1? They work exactly the same with async functions. The return type is what you get AFTER awaiting."

```python
from typing import Optional

# Everything you learned still applies!

async def fetch_user(user_id: int) -> dict:
    """Return type is dict, not Coroutine[dict]"""
    await asyncio.sleep(1)
    return {"id": user_id, "name": "Alice"}

async def find_user(name: str) -> Optional[dict]:
    """Can return dict or None"""
    await asyncio.sleep(1)
    if name == "Alice":
        return {"id": 1, "name": "Alice"}
    return None

async def fetch_many(ids: list[int]) -> list[dict]:
    """Return list of dicts"""
    results = []
    for user_id in ids:
        user = await fetch_user(user_id)
        results.append(user)
    return results
```

**Using Generics with async (Lecture 1 callback):**

```python
from typing import TypeVar, Callable, Awaitable

T = TypeVar("T")

async def with_retry(
    func: Callable[[], Awaitable[T]],  # Async function that returns T
    max_retries: int = 3
) -> T:
    """Generic retry wrapper for async functions"""
    for attempt in range(max_retries):
        try:
            return await func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            await asyncio.sleep(1)
    raise RuntimeError("Unreachable")

# Usage:
async def fetch_data() -> dict:
    ...

result: dict = await with_retry(fetch_data, max_retries=3)
```

---

## 3.5 Error Handling in Async (Connection to Lecture 2)

**Connection to what you've learned:**

> "Remember error handling from Lecture 2? Exceptions work exactly the same in async code. They propagate through await just like through regular calls."

```python
# Your custom exceptions (Lecture 2) work in async too!

class APIError(Exception):
    """Base class for API errors"""
    pass

class RateLimitError(APIError):
    """Too many requests"""
    def __init__(self, retry_after: int):
        self.retry_after = retry_after
        super().__init__(f"Rate limited. Retry after {retry_after}s")

class NotFoundError(APIError):
    """Resource not found"""
    pass

# Using them in async code:
async def fetch_user(user_id: int) -> dict:
    await asyncio.sleep(1)  # Simulate API call
    
    if user_id == 429:
        raise RateLimitError(retry_after=60)
    
    if user_id == 404:
        raise NotFoundError(f"User {user_id} not found")
    
    return {"id": user_id, "name": "Alice"}

# Exception handling — same as sync!
async def main():
    try:
        user = await fetch_user(404)
    except NotFoundError as e:
        print(f"Not found: {e}")
    except RateLimitError as e:
        print(f"Rate limited, waiting {e.retry_after}s")
        await asyncio.sleep(e.retry_after)
        # Retry...
    except APIError as e:
        print(f"API error: {e}")
```

**Exceptions propagate through await:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 EXCEPTION PROPAGATION                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  async def inner():                                             │
│      raise ValueError("oops")                                   │
│                        │                                        │
│  async def middle():   │                                        │
│      await inner() ◀───┘ Exception bubbles up through await    │
│            │                                                    │
│  async def outer():    │                                        │
│      await middle() ◀──┘                                        │
│            │                                                    │
│            ▼                                                    │
│      try:                                                       │
│          ...                                                    │
│      except ValueError:  ← Caught here if you want              │
│          ...                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: CONCURRENT PATTERNS

## 4.1 Sequential vs Concurrent Execution

**First, understand the difference:**

```python
import asyncio
import time

async def fetch(name: str, delay: int) -> str:
    print(f"Starting {name}")
    await asyncio.sleep(delay)
    print(f"Finished {name}")
    return f"{name} result"
```

**Sequential (slow, defeats the purpose):**

```python
async def sequential():
    start = time.time()
    
    # One at a time — each waits for the previous!
    result1 = await fetch("A", 2)
    result2 = await fetch("B", 2)
    result3 = await fetch("C", 2)
    
    print(f"Total time: {time.time() - start:.1f}s")  # 6 seconds!
    return [result1, result2, result3]

asyncio.run(sequential())
```

```
Starting A
Finished A
Starting B
Finished B
Starting C
Finished C
Total time: 6.0s
```

**Why is this still 6 seconds?**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  await fetch("A") ── We wait here until A is done               │
│                      Only THEN do we start B                    │
│  await fetch("B") ── We wait here until B is done               │
│                      Only THEN do we start C                    │
│  await fetch("C") ── We wait here until C is done               │
│                                                                 │
│  Even though we CAN pause, we're not doing anything             │
│  useful during the pause!                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 asyncio.gather() — Running Tasks Together

**`asyncio.gather()` runs multiple coroutines CONCURRENTLY:**

```python
async def concurrent():
    start = time.time()
    
    # All three start at the same time!
    results = await asyncio.gather(
        fetch("A", 2),
        fetch("B", 2),
        fetch("C", 2)
    )
    
    print(f"Total time: {time.time() - start:.1f}s")  # 2 seconds!
    return results

asyncio.run(concurrent())
```

```
Starting A
Starting B
Starting C
Finished A
Finished B
Finished C
Total time: 2.0s
```

**Visualize the difference:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 SEQUENTIAL VS CONCURRENT                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SEQUENTIAL (await one, then await next):                       │
│                                                                 │
│  Time: 0s      2s      4s      6s                               │
│        │       │       │       │                                │
│    A:  [=======]                                                │
│    B:          [=======]                                        │
│    C:                  [=======]                                │
│                                                                 │
│  Total: 6 seconds                                               │
│                                                                 │
│                                                                 │
│  CONCURRENT (gather all):                                       │
│                                                                 │
│  Time: 0s      2s                                               │
│        │       │                                                │
│    A:  [=======]                                                │
│    B:  [=======]                                                │
│    C:  [=======]                                                │
│                                                                 │
│  Total: 2 seconds (time of slowest task)                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How gather works:**

```python
# gather() takes multiple coroutines
results = await asyncio.gather(
    coro1(),  # All start "at once"
    coro2(),
    coro3()
)

# Returns a list of results in the SAME ORDER as inputs
# results[0] = result of coro1
# results[1] = result of coro2
# results[2] = result of coro3
```

**Error handling with gather:**

```python
# By default, if one fails, gather raises immediately
async def might_fail():
    raise ValueError("oops")

try:
    results = await asyncio.gather(
        fetch("A", 2),
        might_fail(),  # This raises!
        fetch("C", 2)
    )
except ValueError:
    print("One task failed")

# Use return_exceptions=True to get errors as results instead
results = await asyncio.gather(
    fetch("A", 2),
    might_fail(),
    fetch("C", 2),
    return_exceptions=True
)
# results = ["A result", ValueError("oops"), "C result"]
```

---

## 4.3 asyncio.create_task() — Fire and Coordinate

**`create_task()` starts a coroutine running in the background:**

```python
async def main():
    # Create tasks — they start running immediately!
    task1 = asyncio.create_task(fetch("A", 2))
    task2 = asyncio.create_task(fetch("B", 2))
    
    print("Tasks are running in background...")
    
    # Do other work while they run
    await asyncio.sleep(0.5)
    print("Half second passed, tasks still running...")
    
    # Now wait for results when you need them
    result1 = await task1
    result2 = await task2
    
    print(f"Results: {result1}, {result2}")
```

**Difference between create_task and gather:**

```
┌─────────────────────────────────────────────────────────────────┐
│           create_task() vs gather()                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  asyncio.gather():                                              │
│  ─────────────────                                              │
│  • Convenience function                                         │
│  • Returns results directly as a list                           │
│  • All-or-nothing (by default)                                  │
│  • Use when: You want all results together                      │
│                                                                 │
│  asyncio.create_task():                                         │
│  ──────────────────────                                         │
│  • Returns a Task object                                        │
│  • Task starts immediately                                      │
│  • Can cancel, check status, add callbacks                      │
│  • Use when: You need more control                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Task control:**

```python
async def main():
    task = asyncio.create_task(fetch("A", 10))
    
    # Check if done
    print(task.done())  # False
    
    # Wait a bit
    await asyncio.sleep(2)
    
    # Cancel it!
    task.cancel()
    
    try:
        result = await task
    except asyncio.CancelledError:
        print("Task was cancelled")
```

---

## 4.4 Real-World Pattern: Fetching Multiple URLs

**The pattern you'll use constantly in backend development:**

```python
import asyncio
from typing import Any

# Simulated async HTTP client (you'll use httpx in Week 4)
async def http_get(url: str) -> dict:
    """Simulate fetching a URL"""
    print(f"Fetching {url}...")
    await asyncio.sleep(1)  # Simulate network delay
    return {"url": url, "status": 200, "data": f"Data from {url}"}

async def fetch_all_prices() -> dict[str, Any]:
    """Fetch prices from multiple exchanges concurrently"""
    
    urls = [
        "https://api.binance.com/btc",
        "https://api.coinbase.com/btc",
        "https://api.kraken.com/btc",
    ]
    
    # Fetch all concurrently
    results = await asyncio.gather(*[http_get(url) for url in urls])
    
    # Process results
    prices = {r["url"]: r["data"] for r in results}
    return prices

async def main():
    import time
    start = time.time()
    
    prices = await fetch_all_prices()
    
    print(f"\nFetched {len(prices)} prices in {time.time() - start:.1f}s")
    for url, data in prices.items():
        print(f"  {url}: {data}")

asyncio.run(main())
```

Output:
```
Fetching https://api.binance.com/btc...
Fetching https://api.coinbase.com/btc...
Fetching https://api.kraken.com/btc...

Fetched 3 prices in 1.0s
  https://api.binance.com/btc: Data from https://api.binance.com/btc
  https://api.coinbase.com/btc: Data from https://api.coinbase.com/btc
  https://api.kraken.com/btc: Data from https://api.kraken.com/btc
```

**Pattern with error handling (production-ready):**

```python
async def fetch_with_fallback(urls: list[str]) -> list[dict]:
    """Fetch multiple URLs, handling individual failures gracefully"""
    
    async def safe_fetch(url: str) -> dict:
        try:
            return await http_get(url)
        except Exception as e:
            return {"url": url, "error": str(e)}
    
    results = await asyncio.gather(*[safe_fetch(url) for url in urls])
    
    # Separate successes and failures
    successes = [r for r in results if "error" not in r]
    failures = [r for r in results if "error" in r]
    
    if failures:
        print(f"Warning: {len(failures)} requests failed")
    
    return successes
```

---

# PART 5: WHEN TO USE ASYNC

## 5.1 The Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                 SHOULD I USE ASYNC?                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      START HERE                                 │
│                          │                                      │
│                          ▼                                      │
│               ┌─────────────────────┐                           │
│               │ Is my code mostly   │                           │
│               │ waiting for I/O?    │                           │
│               └──────────┬──────────┘                           │
│                    │           │                                │
│                   YES          NO                               │
│                    │           │                                │
│                    ▼           ▼                                │
│         ┌──────────────┐  ┌──────────────┐                      │
│         │ Do I need to │  │ Is it CPU-   │                      │
│         │ do multiple  │  │ intensive?   │                      │
│         │ I/O ops?     │  └──────┬───────┘                      │
│         └──────┬───────┘         │                              │
│              │    │           YES│                              │
│            YES    NO             │                              │
│              │    │              ▼                              │
│              ▼    │     ┌──────────────┐                        │
│    ┌────────────┐ │     │ Use multi-   │                        │
│    │ USE ASYNC! │ │     │ processing   │                        │
│    │ ✅         │ │     │ (not async)  │                        │
│    └────────────┘ │     └──────────────┘                        │
│                   │                                             │
│                   ▼                                             │
│          ┌──────────────┐                                       │
│          │ Sync is fine │                                       │
│          │ Keep it      │                                       │
│          │ simple       │                                       │
│          └──────────────┘                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 When Async Helps (I/O-Bound)

**Async SHINES when:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   ASYNC WINS HERE ✅                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEB SERVERS (FastAPI, Starlette)                               │
│  ├─ Handling 1000s of concurrent HTTP requests                  │
│  ├─ Each request waits for database, external APIs              │
│  └─ Async lets one thread handle many connections               │
│                                                                 │
│  API CLIENTS                                                    │
│  ├─ Fetching from multiple APIs simultaneously                  │
│  ├─ Price aggregation from 10 exchanges at once                 │
│  └─ 10 sequential calls = 10x slower than concurrent            │
│                                                                 │
│  CHAT/REAL-TIME APPLICATIONS                                    │
│  ├─ WebSocket connections (thousands open at once)              │
│  ├─ Each connection mostly idle, waiting for messages           │
│  └─ Async handles idle connections efficiently                  │
│                                                                 │
│  DATABASE OPERATIONS                                            │
│  ├─ Multiple queries that don't depend on each other            │
│  ├─ Waiting for Postgres while doing other work                 │
│  └─ Connection pooling with async drivers                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 When Async Hurts (CPU-Bound)

**Async DOES NOT HELP when:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   ASYNC LOSES HERE ❌                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IMAGE/VIDEO PROCESSING                                         │
│  ├─ Resizing 1000 images                                        │
│  ├─ CPU is 100% busy, no waiting                                │
│  └─ Use: multiprocessing, or external tools                     │
│                                                                 │
│  MACHINE LEARNING                                               │
│  ├─ Training models                                             │
│  ├─ Matrix multiplication = pure CPU work                       │
│  └─ Use: GPU acceleration, multiprocessing                      │
│                                                                 │
│  COMPRESSION/ENCRYPTION                                         │
│  ├─ Compressing large files                                     │
│  ├─ CPU-intensive algorithms                                    │
│  └─ Use: run in thread pool or subprocess                       │
│                                                                 │
│  HEAVY COMPUTATION                                              │
│  ├─ Scientific simulations                                      │
│  ├─ No I/O to wait for                                          │
│  └─ Use: numpy, multiprocessing, Cython                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why async doesn't help CPU-bound:**

```
CPU-BOUND WORK:

  Async version:
    Task A: [CALCULATE====][CALCULATE====][CALCULATE====]
    Task B:                                              [CALCULATE...]
    
    Total: Still 6 seconds. Tasks CAN'T overlap because 
           the CPU is always busy.

  What you need instead:
    Process 1: [CALCULATE====]
    Process 2: [CALCULATE====]   ← True parallelism
    Process 3: [CALCULATE====]
    
    Total: 2 seconds (with 3 CPU cores)
```

---

## 5.4 Common Mistakes and Misconceptions

### Mistake 1: Using blocking calls in async code

```python
# ❌ WRONG: time.sleep() blocks the entire event loop!
async def bad_sleep():
    print("Starting")
    time.sleep(2)  # BLOCKS EVERYTHING
    print("Done")

# ✅ CORRECT: asyncio.sleep() yields control
async def good_sleep():
    print("Starting")
    await asyncio.sleep(2)  # Other tasks can run
    print("Done")
```

**Demonstration of the problem:**

```python
import asyncio
import time

async def task(name: str):
    print(f"{name} starting")
    time.sleep(2)  # ❌ Blocking!
    print(f"{name} done")

async def main():
    await asyncio.gather(task("A"), task("B"), task("C"))

# This takes 6 seconds, not 2! The blocking sleep ruins concurrency.
asyncio.run(main())
```

---

### Mistake 2: Forgetting to await

```python
# ❌ WRONG: Forgot await — coroutine never runs!
async def main():
    fetch_data()  # Returns coroutine object, doesn't run!
    # Python warns: "coroutine 'fetch_data' was never awaited"

# ✅ CORRECT: Actually await it
async def main():
    await fetch_data()  # Runs the coroutine
```

---

### Mistake 3: Calling asyncio.run() inside async function

```python
# ❌ WRONG: Can't nest asyncio.run()
async def outer():
    asyncio.run(inner())  # RuntimeError: event loop already running

# ✅ CORRECT: Use await inside async functions
async def outer():
    await inner()
```

---

### Mistake 4: Thinking async is always faster

```python
# For simple, single operations, async adds overhead
# ❌ Unnecessary complexity:
async def fetch_one_thing():
    result = await http_get("url")
    return result

# ✅ If you only need one thing sequentially, sync is fine:
def fetch_one_thing():
    result = requests.get("url")
    return result.json()
```

**Rule of thumb:**

> "Use async when you have MULTIPLE I/O operations that can run concurrently. For single sequential operations, sync code is simpler and fine."

---

### Mistake 5: Not understanding that async is still single-threaded

```python
# This CPU-intensive work BLOCKS the event loop!
async def bad_compute():
    # ❌ This freezes all other tasks for 5 seconds
    total = sum(range(100_000_000))  # CPU-bound!
    return total

# ✅ For CPU-bound work, use run_in_executor
async def good_compute():
    loop = asyncio.get_event_loop()
    total = await loop.run_in_executor(
        None,  # Use default thread pool
        lambda: sum(range(100_000_000))
    )
    return total
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                    ASYNC QUICK REFERENCE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DEFINE A COROUTINE:                                            │
│      async def my_function() -> str:                            │
│          return "result"                                        │
│                                                                 │
│  CALL A COROUTINE (inside async function):                      │
│      result = await my_function()                               │
│                                                                 │
│  START THE EVENT LOOP (entry point):                            │
│      asyncio.run(main())                                        │
│                                                                 │
│  RUN MULTIPLE CONCURRENTLY:                                     │
│      results = await asyncio.gather(coro1(), coro2(), coro3())  │
│                                                                 │
│  START BACKGROUND TASK:                                         │
│      task = asyncio.create_task(my_coro())                      │
│      # ... do other work ...                                    │
│      result = await task                                        │
│                                                                 │
│  SLEEP (non-blocking):                                          │
│      await asyncio.sleep(1.0)                                   │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ time.sleep()        → Use asyncio.sleep()               │
│      ❌ requests.get()      → Use httpx.AsyncClient (Week 4)    │
│      ❌ forgot await        → Coroutine never runs              │
│      ❌ asyncio.run() twice → Use gather/create_task instead    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ASYNC = SMART WAITING                                          │
│                                                                 │
│  When your code waits for I/O (network, disk, database),        │
│  async lets other code run during that wait.                    │
│                                                                 │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐                    │
│  │ Waiting │ ──▶ │ Event   │ ──▶ │ Another │                    │
│  │ Task    │     │ Loop    │     │ Task    │                    │
│  └─────────┘     └─────────┘     └─────────┘                    │
│       │               │               │                         │
│       │               │               │                         │
│       └───────────────┴───────────────┘                         │
│                       │                                         │
│               Cooperative multitasking                          │
│               (they take turns nicely)                          │
│                                                                 │
│  THE RESTAURANT ANALOGY:                                        │
│  ├─ Event Loop = Restaurant Manager                             │
│  ├─ Coroutines = Waiters who can pause mid-task                 │
│  ├─ await = "I'll wait for the kitchen, help other tables"      │
│  └─ gather = "Take all orders first, then deliver as ready"     │
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
│  WEEK 1 (Next lecture):                                         │
│  └─ Git workflow                                                │
│                                                                 │
│  WEEK 1 MINI-PROJECT:                                           │
│  └─ Async CLI tool that fetches crypto prices                   │
│     You'll USE everything from this lecture!                    │
│                                                                 │
│  WEEK 2 (FastAPI):                                              │
│  └─ FastAPI uses async for all route handlers                   │
│     async def get_user() → your knowledge transfers             │
│                                                                 │
│  WEEK 4 (External APIs):                                        │
│  └─ httpx AsyncClient for concurrent API requests               │
│     You'll fetch from multiple exchanges simultaneously         │
│                                                                 │
│  WEEK 8 (Celery):                                               │
│  └─ Compare async to background workers                         │
│     Different tools for different problems                      │
│                                                                 │
│  WEEK 9 (WebSockets):                                           │
│  └─ Real-time connections use async extensively                 │
│     Handle thousands of concurrent connections                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---
