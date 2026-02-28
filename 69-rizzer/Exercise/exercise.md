# LECTURE DESIGN PHILOSOPHY 

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

# LECTURE OUTLINE

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

**`create_task()` schedules a coroutine to run "in the background":**

```
┌─────────────────────────────────────────────────────────────────┐
│                    create_task() CONCEPT                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RESTAURANT ANALOGY:                                            │
│                                                                 │
│  await coro() = "Go to kitchen, wait for food, bring it back"   │
│                 (You personally wait)                           │
│                                                                 │
│  create_task() = "Tell kitchen to start cooking, I'll check     │
│                   back later when I need it"                    │
│                 (Kitchen works while you do other things)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Basic usage:**

```python
import asyncio

async def cook_dish(name: str, time_needed: int) -> str:
    print(f"🍳 Started cooking {name}")
    await asyncio.sleep(time_needed)
    print(f"✅ {name} is ready!")
    return f"{name} (cooked)"

async def main():
    # Create task — starts running IMMEDIATELY in background
    task = asyncio.create_task(cook_dish("Pasta", 3))
    
    print(f"Task created: {task}")
    print(f"Is it done? {task.done()}")  # False — still cooking
    
    # Do other work while pasta cooks
    print("Meanwhile, I'm setting the table...")
    await asyncio.sleep(1)
    print("Table is set!")
    
    print(f"Is pasta done? {task.done()}")  # Still False
    
    # Now wait for the result when we need it
    result = await task
    print(f"Got result: {result}")

asyncio.run(main())
```

Output:
```
Task created: <Task pending name='Task-2' coro=<cook_dish() running at ...>>
Is it done? False
🍳 Started cooking Pasta
Meanwhile, I'm setting the table...
Table is set!
Is pasta done? False
✅ Pasta is ready!
Got result: Pasta (cooked)
```

---

**Task lifecycle:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    TASK LIFECYCLE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   create_task(coro())                                           │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                               │
│   │   PENDING   │  ← Task is scheduled, may or may not be       │
│   │             │    running yet                                │
│   └──────┬──────┘                                               │
│          │                                                      │
│          │ (runs until completion, exception, or cancel)        │
│          │                                                      │
│    ┌─────┴─────┬─────────────┐                                  │
│    ▼           ▼             ▼                                  │
│ ┌──────┐  ┌─────────┐  ┌───────────┐                            │
│ │ DONE │  │ FAILED  │  │ CANCELLED │                            │
│ │      │  │         │  │           │                            │
│ └──────┘  └─────────┘  └───────────┘                            │
│                                                                 │
│  task.done()      → True if in any final state                  │
│  task.cancelled() → True if cancelled                           │
│  task.result()    → Get result (raises if failed/cancelled)     │
│  task.exception() → Get exception (None if succeeded)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

**Task methods and properties:**

```python
async def demonstrate_task_api():
    async def slow_task():
        await asyncio.sleep(2)
        return "completed"
    
    task = asyncio.create_task(slow_task(), name="my-slow-task")
    
    # ═══════════════════════════════════════════════════════════
    # INSPECTION METHODS
    # ═══════════════════════════════════════════════════════════
    
    print(task.get_name())      # "my-slow-task"
    print(task.done())          # False (still running)
    print(task.cancelled())     # False (not cancelled)
    
    # ═══════════════════════════════════════════════════════════
    # WAITING FOR COMPLETION
    # ═══════════════════════════════════════════════════════════
    
    result = await task         # Wait and get result
    print(task.done())          # True (now complete)
    print(task.result())        # "completed"
    
asyncio.run(demonstrate_task_api())
```

---

**Cancelling tasks:**

```python
async def cancellation_demo():
    async def long_running_task():
        try:
            print("Task started, will take 10 seconds...")
            await asyncio.sleep(10)
            return "finished"
        except asyncio.CancelledError:
            print("Task was cancelled! Cleaning up...")
            # Perform cleanup here
            raise  # Re-raise to properly mark as cancelled
    
    task = asyncio.create_task(long_running_task())
    
    # Let it run for 2 seconds
    await asyncio.sleep(2)
    
    # Cancel it!
    print("Cancelling task...")
    task.cancel()
    
    # Wait for cancellation to complete
    try:
        await task
    except asyncio.CancelledError:
        print("Task cancellation confirmed")
    
    print(f"Task state - done: {task.done()}, cancelled: {task.cancelled()}")

asyncio.run(cancellation_demo())
```

Output:
```
Task started, will take 10 seconds...
Cancelling task...
Task was cancelled! Cleaning up...
Task cancellation confirmed
Task state - done: True, cancelled: True
```

---

**Pattern: Managing multiple tasks:**

```python
async def fetch_price(exchange: str) -> dict:
    """Simulate fetching price from an exchange"""
    await asyncio.sleep(1)
    return {"exchange": exchange, "price": 50000.0}

async def monitor_prices():
    """Create multiple tasks and manage them"""
    
    exchanges = ["binance", "coinbase", "kraken"]
    
    # Create all tasks
    tasks = [
        asyncio.create_task(fetch_price(ex), name=f"fetch-{ex}")
        for ex in exchanges
    ]
    
    print(f"Created {len(tasks)} tasks")
    
    # Wait for all to complete
    results = []
    for task in tasks:
        result = await task
        results.append(result)
        print(f"Task '{task.get_name()}' completed: {result}")
    
    return results

asyncio.run(monitor_prices())
```

---

**create_task() vs gather() — When to use which:**

```
┌─────────────────────────────────────────────────────────────────┐
│             create_task() vs gather()                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE gather() WHEN:                                             │
│  ──────────────────                                             │
│  • You want all results at once                                 │
│  • Simple "run these together, give me results"                 │
│  • You don't need to cancel individual tasks                    │
│                                                                 │
│      results = await asyncio.gather(                            │
│          fetch("A"),                                            │
│          fetch("B"),                                            │
│          fetch("C")                                             │
│      )                                                          │
│      # results = ["A result", "B result", "C result"]           │
│                                                                 │
│                                                                 │
│  USE create_task() WHEN:                                        │
│  ───────────────────────                                        │
│  • You need to do work WHILE tasks run                          │
│  • You need to cancel specific tasks                            │
│  • You want to check task status                                │
│  • You need task references for later                           │
│  • Tasks have different priorities/handling                     │
│                                                                 │
│      task_a = asyncio.create_task(fetch("A"))                   │
│      task_b = asyncio.create_task(fetch("B"))                   │
│                                                                 │
│      # Do other work...                                         │
│      await do_something_else()                                  │
│                                                                 │
│      # Maybe cancel one based on condition                      │
│      if should_cancel:                                          │
│          task_b.cancel()                                        │
│                                                                 │
│      result_a = await task_a                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

**Pattern: Timeout with create_task:**

```python
async def fetch_with_timeout(url: str, timeout: float) -> dict | None:
    """Fetch with a timeout — cancel if too slow"""
    
    async def do_fetch():
        await asyncio.sleep(5)  # Simulate slow request
        return {"url": url, "data": "..."}
    
    task = asyncio.create_task(do_fetch())
    
    try:
        # Wait for task OR timeout, whichever comes first
        result = await asyncio.wait_for(task, timeout=timeout)
        return result
    except asyncio.TimeoutError:
        print(f"Fetch timed out after {timeout}s, cancelling...")
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            pass
        return None

async def main():
    result = await fetch_with_timeout("https://slow-api.com", timeout=2.0)
    print(f"Result: {result}")

asyncio.run(main())
```

Output:
```
Fetch timed out after 2.0s, cancelling...
Result: None
```

---

## 4.4 asyncio.Queue — Producer-Consumer Pattern

**The Queue is for passing data between coroutines:**

```
┌──────────────────────────────────────────────────────────────────┐
│                    asyncio.Queue CONCEPT                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RESTAURANT ANALOGY:                                             │
│                                                                  │
│  The Queue is the ORDER WINDOW between waiters and kitchen.      │
│                                                                  │
│  ┌───────────┐               ┌───────┐              ┌───────────┐│           
│  │ WAITERS   │ ──── put ───▶ |[QUEUE]| ──── get ──▶ │ KITCHEN   ││
|  |(producers)|    orders     │ 📋📋📋 │    orders    │(consumers)│                   
│  └───────────┘               └───────┘              └───────────┘│                                  
│                                                                  │
│                                                                  │
│  • Waiters PUT orders into the queue                             │
│  • Kitchen GETs orders from the queue                            │
│  • Queue handles coordination automatically                      │
│  • If kitchen is slow, orders wait in queue                      │
│  • If no orders, kitchen waits                                   │ 
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

**Basic Queue usage:**

```python
import asyncio

async def producer(queue: asyncio.Queue, name: str):
    """Produces items and puts them in the queue"""
    for i in range(3):
        item = f"{name}-item-{i}"
        await queue.put(item)
        print(f"📤 {name} produced: {item}")
        await asyncio.sleep(0.5)  # Simulate work
    print(f"✅ {name} finished producing")

async def consumer(queue: asyncio.Queue, name: str):
    """Consumes items from the queue"""
    while True:
        item = await queue.get()  # Waits if queue is empty
        print(f"📥 {name} consumed: {item}")
        await asyncio.sleep(1)  # Simulate processing
        queue.task_done()  # Mark item as processed

async def main():
    queue = asyncio.Queue()
    
    # Create producer and consumer tasks
    producer_task = asyncio.create_task(producer(queue, "Producer-1"))
    consumer_task = asyncio.create_task(consumer(queue, "Consumer-1"))
    
    # Wait for producer to finish
    await producer_task
    
    # Wait for all items to be processed
    await queue.join()  # Blocks until all task_done() called
    
    # Cancel consumer (it's in infinite loop)
    consumer_task.cancel()
    print("All items processed!")

asyncio.run(main())
```

Output:
```
📤 Producer-1 produced: Producer-1-item-0
📥 Consumer-1 consumed: Producer-1-item-0
📤 Producer-1 produced: Producer-1-item-1
📤 Producer-1 produced: Producer-1-item-2
✅ Producer-1 finished producing
📥 Consumer-1 consumed: Producer-1-item-1
📥 Consumer-1 consumed: Producer-1-item-2
All items processed!
```

---

**Queue methods:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    QUEUE METHODS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CREATING:                                                      │
│  ─────────                                                      │
│  queue = asyncio.Queue()           # Unlimited size             │
│  queue = asyncio.Queue(maxsize=10) # Max 10 items               │
│                                                                 │
│  ADDING ITEMS:                                                  │
│  ─────────────                                                  │
│  await queue.put(item)      # Wait if full (maxsize)            │
│  queue.put_nowait(item)     # Raise QueueFull if full           │
│                                                                 │
│  GETTING ITEMS:                                                 │
│  ──────────────                                                 │
│  item = await queue.get()   # Wait if empty                     │
│  item = queue.get_nowait()  # Raise QueueEmpty if empty         │
│                                                                 │
│  TRACKING COMPLETION:                                           │
│  ────────────────────                                           │
│  queue.task_done()          # Mark item as processed            │
│  await queue.join()         # Wait until all items processed    │
│                                                                 │
│  INSPECTION:                                                    │
│  ───────────                                                    │
│  queue.qsize()              # Current number of items           │
│  queue.empty()              # True if empty                     │
│  queue.full()               # True if at maxsize                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

**Pattern: Multiple producers and consumers:**

```python
import asyncio
import random

async def producer(queue: asyncio.Queue, producer_id: int, num_items: int):
    """Simulate producing work items"""
    for i in range(num_items):
        # Simulate creating work (e.g., receiving API requests)
        await asyncio.sleep(random.uniform(0.1, 0.3))
        
        item = {"producer": producer_id, "item": i, "data": f"work-{i}"}
        await queue.put(item)
        print(f"📤 Producer {producer_id} added item {i} (queue size: {queue.qsize()})")

async def consumer(queue: asyncio.Queue, consumer_id: int):
    """Simulate processing work items"""
    processed = 0
    while True:
        try:
            item = await queue.get()
            
            # Simulate processing work (e.g., database write)
            await asyncio.sleep(random.uniform(0.2, 0.5))
            
            print(f"📥 Consumer {consumer_id} processed: {item}")
            processed += 1
            queue.task_done()
            
        except asyncio.CancelledError:
            print(f"🛑 Consumer {consumer_id} stopped (processed {processed} items)")
            raise

async def main():
    queue = asyncio.Queue(maxsize=5)  # Bounded queue
    
    # Create multiple producers and consumers
    producers = [
        asyncio.create_task(producer(queue, i, 3))
        for i in range(2)
    ]
    
    consumers = [
        asyncio.create_task(consumer(queue, i))
        for i in range(3)
    ]
    
    # Wait for all producers to finish
    await asyncio.gather(*producers)
    print("\n--- All producers done ---\n")
    
    # Wait for queue to be fully processed
    await queue.join()
    print("\n--- All items processed ---\n")
    
    # Cancel consumers
    for c in consumers:
        c.cancel()
    
    # Wait for consumers to acknowledge cancellation
    await asyncio.gather(*consumers, return_exceptions=True)

asyncio.run(main())
```

---

**Real-world pattern: Rate-limited API fetcher:**

```python
import asyncio
from typing import Any

async def rate_limited_fetcher(
    urls: list[str],
    max_concurrent: int = 3
) -> list[dict]:
    """
    Fetch multiple URLs with concurrency limit.
    Uses Queue to control how many requests run at once.
    """
    
    queue: asyncio.Queue[str] = asyncio.Queue()
    results: list[dict] = []
    results_lock = asyncio.Lock()
    
    # Add all URLs to queue
    for url in urls:
        await queue.put(url)
    
    async def worker(worker_id: int):
        while True:
            try:
                url = queue.get_nowait()
            except asyncio.QueueEmpty:
                break
            
            print(f"Worker {worker_id} fetching: {url}")
            
            # Simulate API call
            await asyncio.sleep(1)
            result = {"url": url, "status": 200}
            
            async with results_lock:
                results.append(result)
            
            queue.task_done()
    
    # Create limited number of workers
    workers = [
        asyncio.create_task(worker(i))
        for i in range(max_concurrent)
    ]
    
    await asyncio.gather(*workers)
    return results

async def main():
    urls = [f"https://api.example.com/item/{i}" for i in range(10)]
    
    print(f"Fetching {len(urls)} URLs with max 3 concurrent requests\n")
    
    import time
    start = time.time()
    results = await rate_limited_fetcher(urls, max_concurrent=3)
    elapsed = time.time() - start
    
    print(f"\nFetched {len(results)} URLs in {elapsed:.1f}s")
    # With 10 URLs, 3 concurrent, 1s each: ~4 seconds total

asyncio.run(main())
```

---

**Queue visualization:**

```
┌─────────────────────────────────────────────────────────────────┐
│              PRODUCER-CONSUMER FLOW                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│   Producer 1 ──┐                        ┌──▶ Consumer 1         │
│                │     ┌───────────┐      │                       │
│   Producer 2 ──┼────▶│  QUEUE    │──────┼──▶ Consumer 2         │
│                │     │ [A][B][C] │      │                       │
│   Producer 3 ──┘     └───────────┘      └──▶ Consumer 3         │
│                                                                 │
│                                                                 │
│   WITH maxsize=3:                                               │
│   ─────────────────                                             │
│                                                                 │
│   Queue: [A][B][C]  ← FULL                                      │
│                                                                 │
│   Producer calls: await queue.put(D)                            │
│                   └─▶ WAITS until consumer takes one            │
│                                                                 │
│   Consumer calls: await queue.get()                             │
│                   └─▶ Returns A, queue is now [B][C]            │
│                                                                 │
│   Producer: put(D) completes, queue is [B][C][D]                │
│                                                                 │
│                                                                 │
│   WITH empty queue:                                             │
│   ─────────────────                                             │
│                                                                 │
│   Queue: []  ← EMPTY                                            │
│                                                                 │
│   Consumer calls: await queue.get()                             │
│                   └─▶ WAITS until producer adds one             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 asyncio.Event — Signaling Between Coroutines

**Event is a simple signal flag:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    asyncio.Event CONCEPT                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RESTAURANT ANALOGY:                                            │
│                                                                 │
│  The Event is the "OPEN/CLOSED" sign on the kitchen.            │
│                                                                 │
│  • Waiters: "await event.wait()"                                │
│    └─ "I'll wait here until kitchen opens"                      │
│                                                                 │
│  • Kitchen: "event.set()"                                       │
│    └─ "Kitchen is now OPEN!" (flips sign)                       │
│                                                                 │
│  • Manager: "event.clear()"                                     │
│    └─ "Kitchen is now CLOSED" (flips sign back)                 │
│                                                                 │
│  Multiple waiters can wait on the same event.                   │
│  When set(), ALL waiters are released at once.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

**Basic Event usage:**

```python
import asyncio

async def waiter(event: asyncio.Event, name: str):
    """Wait for the event to be set"""
    print(f"⏳ {name} is waiting for the signal...")
    await event.wait()  # Blocks until event.set() is called
    print(f"✅ {name} received the signal! Proceeding...")

async def signaler(event: asyncio.Event):
    """Set the event after some work"""
    print("🔧 Signaler doing preparation work...")
    await asyncio.sleep(2)
    print("📢 Signaler: Setting the event!")
    event.set()  # Release all waiters

async def main():
    event = asyncio.Event()
    
    print(f"Event state: {event.is_set()}")  # False
    
    # Start multiple waiters
    await asyncio.gather(
        waiter(event, "Waiter-1"),
        waiter(event, "Waiter-2"),
        waiter(event, "Waiter-3"),
        signaler(event)
    )
    
    print(f"Event state: {event.is_set()}")  # True

asyncio.run(main())
```

Output:
```
Event state: False
⏳ Waiter-1 is waiting for the signal...
⏳ Waiter-2 is waiting for the signal...
⏳ Waiter-3 is waiting for the signal...
🔧 Signaler doing preparation work...
📢 Signaler: Setting the event!
✅ Waiter-1 received the signal! Proceeding...
✅ Waiter-2 received the signal! Proceeding...
✅ Waiter-3 received the signal! Proceeding...
Event state: True
```

---

**Event methods:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    EVENT METHODS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  event = asyncio.Event()                                        │
│                                                                 │
│  event.set()        # Set the flag to True                      │
│                     # All waiters are released immediately      │
│                                                                 │
│  event.clear()      # Set the flag to False                     │
│                     # Future wait() calls will block again      │
│                                                                 │
│  event.is_set()     # Check if flag is True                     │
│                                                                 │
│  await event.wait() # Block until flag is True                  │
│                     # Returns immediately if already True       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

**Pattern: Initialization gate:**

```python
import asyncio

class DatabaseConnection:
    """Simulated database that takes time to connect"""
    
    def __init__(self):
        self._ready = asyncio.Event()
        self._connected = False
    
    async def connect(self):
        """Simulate slow database connection"""
        print("🔌 Database connecting...")
        await asyncio.sleep(2)  # Simulate connection time
        self._connected = True
        self._ready.set()  # Signal that we're ready
        print("✅ Database connected!")
    
    async def query(self, sql: str) -> list:
        """Execute a query (waits for connection first)"""
        await self._ready.wait()  # Wait until connected
        print(f"📊 Executing: {sql}")
        await asyncio.sleep(0.1)
        return [{"result": "data"}]

async def worker(db: DatabaseConnection, worker_id: int):
    """Worker that needs the database"""
    print(f"Worker {worker_id} wants to query...")
    result = await db.query(f"SELECT * FROM table_{worker_id}")
    print(f"Worker {worker_id} got result: {result}")

async def main():
    db = DatabaseConnection()
    
    # Start workers and connection concurrently
    # Workers will wait for connection to be ready
    await asyncio.gather(
        db.connect(),
        worker(db, 1),
        worker(db, 2),
        worker(db, 3),
    )

asyncio.run(main())
```

Output:
```
🔌 Database connecting...
Worker 1 wants to query...
Worker 2 wants to query...
Worker 3 wants to query...
✅ Database connected!
📊 Executing: SELECT * FROM table_1
📊 Executing: SELECT * FROM table_2
📊 Executing: SELECT * FROM table_3
Worker 1 got result: [{'result': 'data'}]
Worker 2 got result: [{'result': 'data'}]
Worker 3 got result: [{'result': 'data'}]
```

---

**Pattern: Graceful shutdown:**

```python
import asyncio

async def worker(shutdown_event: asyncio.Event, worker_id: int):
    """Worker that runs until shutdown is signaled"""
    print(f"🟢 Worker {worker_id} started")
    
    work_count = 0
    while not shutdown_event.is_set():
        # Do some work
        print(f"   Worker {worker_id} doing work #{work_count}")
        work_count += 1
        
        # Check for shutdown every 0.5 seconds
        try:
            await asyncio.wait_for(shutdown_event.wait(), timeout=0.5)
        except asyncio.TimeoutError:
            continue  # No shutdown signal, keep working
    
    print(f"🔴 Worker {worker_id} shutting down (did {work_count} tasks)")

async def main():
    shutdown_event = asyncio.Event()
    
    # Start workers
    workers = [
        asyncio.create_task(worker(shutdown_event, i))
        for i in range(3)
    ]
    
    # Let them work for 2 seconds
    await asyncio.sleep(2)
    
    # Signal shutdown
    print("\n📢 Signaling shutdown...\n")
    shutdown_event.set()
    
    # Wait for all workers to finish
    await asyncio.gather(*workers)
    print("\n✅ All workers shut down cleanly")

asyncio.run(main())
```

---

**Pattern: Periodic reset (reusable gate):**

```python
import asyncio

async def batch_processor(
    batch_ready: asyncio.Event,
    items_queue: asyncio.Queue
):
    """Process items in batches when signaled"""
    while True:
        # Wait for batch to be ready
        await batch_ready.wait()
        
        # Process all items in queue
        batch = []
        while not items_queue.empty():
            try:
                item = items_queue.get_nowait()
                batch.append(item)
            except asyncio.QueueEmpty:
                break
        
        if batch:
            print(f"📦 Processing batch of {len(batch)} items: {batch}")
            await asyncio.sleep(0.5)  # Simulate processing
        
        # Clear event for next batch
        batch_ready.clear()

async def item_collector(
    batch_ready: asyncio.Event,
    items_queue: asyncio.Queue,
    batch_size: int = 5
):
    """Collect items and signal when batch is ready"""
    item_count = 0
    
    for i in range(12):  # Simulate 12 incoming items
        await asyncio.sleep(0.1)  # Items arrive over time
        
        await items_queue.put(f"item-{i}")
        item_count += 1
        print(f"📥 Collected item-{i} ({item_count}/{batch_size})")
        
        if item_count >= batch_size:
            print("📢 Batch ready!")
            batch_ready.set()
            item_count = 0
            await asyncio.sleep(0.6)  # Wait for processing
    
    # Final batch (if any remaining)
    if item_count > 0:
        batch_ready.set()

async def main():
    batch_ready = asyncio.Event()
    items_queue = asyncio.Queue()
    
    processor = asyncio.create_task(
        batch_processor(batch_ready, items_queue)
    )
    
    await item_collector(batch_ready, items_queue, batch_size=5)
    
    # Wait for final batch
    await asyncio.sleep(1)
    processor.cancel()

asyncio.run(main())
```

---

**Comparison: When to use what:**

```
┌─────────────────────────────────────────────────────────────────┐
│           CHOOSING THE RIGHT PRIMITIVE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE create_task() WHEN:                                        │
│  • You want to start work in background                         │
│  • You need to cancel/monitor individual operations             │
│  • Tasks are independent, don't share data directly             │
│                                                                 │
│  USE Queue WHEN:                                                │
│  • You have producer-consumer pattern                           │
│  • Multiple workers processing from same work pool              │
│  • You need backpressure (maxsize limits producers)             │
│  • Data flows FROM producers TO consumers                       │
│                                                                 │
│  USE Event WHEN:                                                │
│  • You need to signal "something happened"                      │
│  • Multiple coroutines wait for same condition                  │
│  • Initialization/shutdown coordination                         │
│  • No data transfer, just signaling                             │
│                                                                 │
│                                                                 │
│  COMBINATIONS:                                                  │
│  ─────────────                                                  │
│  • Task + Event: Task waits for initialization                  │
│  • Task + Queue: Workers (tasks) consume from queue             │
│  • Queue + Event: Batch processing with batch-ready signal      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.6 Real-World Pattern: Fetching Multiple URLs

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
## Updated Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│               ASYNC PRIMITIVES QUICK REFERENCE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TASKS (create_task):                                           │
│  ─────────────────────                                          │
│  task = asyncio.create_task(coro(), name="my-task")             │
│  task.done()           # Check if complete                      │
│  task.cancel()         # Request cancellation                   │
│  task.result()         # Get result (after done)                │
│  result = await task   # Wait for completion                    │
│                                                                 │
│  QUEUE:                                                         │
│  ──────                                                         │
│  queue = asyncio.Queue(maxsize=10)                              │
│  await queue.put(item)   # Add item (waits if full)             │
│  item = await queue.get()# Get item (waits if empty)            │
│  queue.task_done()       # Mark item processed                  │
│  await queue.join()      # Wait for all items processed         │
│                                                                 │
│  EVENT:                                                         │
│  ──────                                                         │
│  event = asyncio.Event()                                        │
│  event.set()             # Set flag True (release waiters)      │
│  event.clear()           # Set flag False                       │
│  event.is_set()          # Check flag                           │
│  await event.wait()      # Wait until flag is True              │
│                                                                 │
│  PATTERNS:                                                      │
│  ─────────                                                      │
│  • Background work     → create_task()                          │
│  • Producer-consumer   → Queue                                  │
│  • Wait for condition  → Event                                  │
│  • Run together        → gather() or multiple create_task()     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
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

# Quick Reference Card

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

# Summary: The Key Mental Model

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

# Connection to Upcoming Lectures

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

Ready for your exercise template!