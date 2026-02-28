# PHASE 1: TOPIC ANALYSIS

## 1. The Main Opponent (Prior Mental Model)

**Naive Belief:** "Code executes line by line. Adding `async/await` magically makes things run in parallel."

Learners arrive with a **synchronous, sequential execution** mental model. They believe:
- `async def` automatically parallelizes the function
- Adding `await` before each call is sufficient for concurrency
- Async Python uses multiple CPU cores
- Async always makes code faster

**The Prediction Gap:** They expect three `await` calls to run simultaneously, when in reality sequential `await` statements run one after another — just like synchronous code.

## 2. The Main Purpose (Essence)

**Core Insight:** "When you're waiting for I/O, let other tasks run instead of blocking."

Async solves **I/O idle time waste**. When your code waits for a network response, the CPU sits idle. Async allows:
- One thread to manage thousands of I/O-bound tasks
- Cooperative yielding at `await` points
- Concurrent (not parallel) execution of waiting operations

**The Key Distinction:** `await` is a **pause point**, not a parallelization command. Concurrency only happens when multiple tasks are **scheduled together** (via `gather` or `create_task`).

---

# PHASE 2: THE EXERCISES

---

## LEVEL 1: PREDICT & EXPLAIN (The Trap)

**Scenario:** A developer writes their first async code to fetch data from three cryptocurrency exchanges. They expect significant speedup.

```python
import asyncio
import time

async def fetch_price(exchange: str) -> dict:
    """Simulate fetching price from an exchange API"""
    print(f"[{time.strftime('%H:%M:%S')}] Starting {exchange}...")
    await asyncio.sleep(2)  # Simulate 2 second API latency
    print(f"[{time.strftime('%H:%M:%S')}] Finished {exchange}")
    return {"exchange": exchange, "price": 50000.0}

async def get_all_prices() -> list[dict]:
    """Fetch prices from all exchanges"""
    binance = await fetch_price("Binance")
    coinbase = await fetch_price("Coinbase")
    kraken = await fetch_price("Kraken")
    return [binance, coinbase, kraken]

async def main():
    start = time.time()
    prices = await get_all_prices()
    elapsed = time.time() - start
    print(f"\nTotal time: {elapsed:.1f} seconds")
    print(f"Fetched {len(prices)} prices")

asyncio.run(main())
```

**Questions:**

1. **Predict the output.** Specifically:
   - What is the total execution time?
   - In what order do the "Starting" and "Finished" messages appear?

2. **Explain why.** The developer expected ~2 seconds total (all three running concurrently). Why does the actual behavior differ from their expectation?

3. **Mental Trace:** Walk through what the event loop is doing at each `await` statement. When does control return to the event loop, and what tasks are available to run?

---

## LEVEL 2: DEBUG & FIX (The Bottleneck)

**Scenario:** A developer "fixed" the Level 1 code to run concurrently. The code runs without errors, but there's still a critical flaw that defeats the purpose of async.

```python
import asyncio
import time
import requests  # Standard library HTTP client

async def fetch_price(exchange: str, symbol: str) -> dict:
    """Fetch cryptocurrency price from exchange API"""
    print(f"Fetching {symbol} from {exchange}...")
    
    # Simulate API endpoint
    url = f"https://api.{exchange.lower()}.com/v1/ticker/{symbol}"
    
    # Make the HTTP request
    await asyncio.sleep(0)  # "Make it async" 
    response = requests.get(url, timeout=5)  # Real HTTP call
    
    print(f"Got response from {exchange}")
    return {"exchange": exchange, "symbol": symbol, "price": response.json()}

async def get_prices_concurrent(exchanges: list[str], symbol: str) -> list[dict]:
    """Fetch from all exchanges concurrently"""
    tasks = [fetch_price(ex, symbol) for ex in exchanges]
    return await asyncio.gather(*tasks)

async def main():
    exchanges = ["Binance", "Coinbase", "Kraken", "Gemini", "Bitstamp"]
    start = time.time()
    
    try:
        prices = await get_prices_concurrent(exchanges, "BTCUSD")
        print(f"\nFetched {len(prices)} prices in {time.time() - start:.1f}s")
    except Exception as e:
        print(f"Error: {e}")

asyncio.run(main())
```

**Tasks:**

1. **Identify the flaw.** This code uses `asyncio.gather()` correctly. Why will the fetches still run sequentially instead of concurrently? What line is the bottleneck?

2. **Explain the mechanism.** What happens to the event loop when `requests.get()` is called? Why doesn't `await asyncio.sleep(0)` before it help?

3. **Fix the code.** Rewrite `fetch_price` to achieve true concurrent I/O. You may use `asyncio.sleep()` to simulate the HTTP call for now (real async HTTP comes in Week 4).

4. **Bonus (Connection to Lecture 2):** Add proper error handling using a custom exception hierarchy. What happens if one exchange fails? How should `gather` be configured?

---

## LEVEL 3: DESIGN UNDER CONSTRAINTS (The Application)

**Scenario:** You're building a price aggregator for a crypto trading CLI tool (the Week 1 mini-project). You have strict requirements:

**Requirements:**
```
┌─────────────────────────────────────────────────────────────────┐
│                     PRICE AGGREGATOR SPEC                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INPUT:  List of 10 exchange names                              │
│  OUTPUT: Dict mapping exchange → price (or error)               │
│                                                                 │
│  CONSTRAINTS:                                                   │
│  ├─ Each API call has 500ms latency (simulated)                 │
│  ├─ Total execution must complete in under 1 second             │
│  ├─ Some exchanges may fail (simulate 20% failure rate)         │
│  ├─ Failed exchanges should return None, not crash the program  │
│  └─ Must use proper type hints (connect to Lecture 1)           │
│                                                                 │
│  MATH CHECK:                                                    │
│  10 exchanges × 500ms = 5000ms sequential (FAILS constraint)    │
│  Concurrent execution: ~500ms (PASSES constraint)               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Starter Code:**

```python
import asyncio
import random
from typing import Optional

# Type alias (Lecture 1 connection)
PriceResult = dict[str, Optional[float]]

EXCHANGES = [
    "Binance", "Coinbase", "Kraken", "Gemini", "Bitstamp",
    "KuCoin", "Huobi", "OKX", "Bybit", "Gate"
]

async def fetch_single_price(exchange: str) -> float:
    """
    Simulate fetching price from one exchange.
    - Takes 500ms
    - 20% chance of raising an exception
    """
    await asyncio.sleep(0.5)  # 500ms latency
    
    if random.random() < 0.2:
        raise ConnectionError(f"{exchange} API unavailable")
    
    # Return simulated price with some variance
    base_price = 50000.0
    variance = random.uniform(-500, 500)
    return base_price + variance


async def aggregate_prices(exchanges: list[str]) -> PriceResult:
    """
    YOUR TASK: Implement this function.
    
    Requirements:
    - Fetch from all exchanges concurrently
    - Handle individual failures gracefully (return None for failed)
    - Return dict mapping exchange name → price (or None if failed)
    - Must complete in under 1 second for 10 exchanges
    """
    # TODO: Your implementation here
    pass


async def main():
    import time
    start = time.time()
    
    results = await aggregate_prices(EXCHANGES)
    elapsed = time.time() - start
    
    print(f"\n{'='*50}")
    print(f"Completed in {elapsed:.2f} seconds")
    print(f"{'='*50}")
    
    successes = {k: v for k, v in results.items() if v is not None}
    failures = {k: v for k, v in results.items() if v is None}
    
    print(f"\nSuccessful: {len(successes)}/{len(results)}")
    for exchange, price in successes.items():
        print(f"  {exchange}: ${price:,.2f}")
    
    if failures:
        print(f"\nFailed: {len(failures)}")
        for exchange in failures:
            print(f"  {exchange}: unavailable")
    
    # Validation
    assert elapsed < 1.0, f"Too slow! {elapsed:.2f}s > 1.0s"
    assert len(results) == 10, f"Missing results: {len(results)}/10"
    print("\n✅ All constraints satisfied!")


if __name__ == "__main__":
    asyncio.run(main())
```

**Your Task:** Implement `aggregate_prices()` to meet all constraints.

---

## LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE (The Distinction)

**Task:** Demonstrate when async is the **WRONG** choice.

**Part A: The CPU-Bound Trap**

A junior developer heard "async makes Python faster" and wrote this image processing code:

```python
import asyncio
import time

def cpu_intensive_hash(data: bytes) -> str:
    """Simulate CPU-intensive hashing (e.g., for proof-of-work)"""
    total = 0
    for i in range(10_000_000):  # Heavy computation
        total += i * len(data)
    return hex(total % (2**32))

async def process_block_async(block_id: int) -> dict:
    """Process a blockchain block 'asynchronously'"""
    data = f"block_{block_id}".encode()
    hash_result = cpu_intensive_hash(data)  # CPU-bound work
    return {"block": block_id, "hash": hash_result}

async def main():
    start = time.time()
    
    # "Run concurrently"
    results = await asyncio.gather(
        process_block_async(1),
        process_block_async(2),
        process_block_async(3),
    )
    
    elapsed = time.time() - start
    print(f"Processed {len(results)} blocks in {elapsed:.1f}s")

asyncio.run(main())
```

**Questions:**

1. **Predict:** Will the three blocks process concurrently? What will the total time be if each `cpu_intensive_hash` takes 2 seconds?

2. **Explain:** Why doesn't `asyncio.gather` help here? What is the event loop doing while `cpu_intensive_hash` runs?

3. **Demonstrate:** Modify the code to PROVE that async provides no benefit. Add timing prints to show sequential execution.

**Part B: The Overhead Trap**

4. **Scenario:** You need to make exactly ONE HTTP request and return the result. Write two versions:
   - A synchronous version using a hypothetical `sync_http_get(url)`
   - An async version using `async_http_get(url)`

5. **Argue:** In this single-request scenario, which version is better? Why might async add unnecessary complexity with no benefit?

---

## LEVEL 5: THE TEACH-BACK (Synthesis)

**Scenario:** You're reviewing a pull request from a junior developer. They've left these comments in the code:

```python
import asyncio
import time

async def fetch_user_data(user_id: int) -> dict:
    """Fetch user from database"""
    await asyncio.sleep(0.5)  # Simulate DB query
    return {"id": user_id, "name": f"User_{user_id}"}

async def fetch_user_orders(user_id: int) -> list:
    """Fetch orders for user"""
    await asyncio.sleep(0.5)  # Simulate DB query  
    return [{"order_id": i, "user_id": user_id} for i in range(3)]

async def get_user_profile(user_id: int) -> dict:
    # Junior's comment: "I made this async so it runs on multiple CPU cores
    # and processes requests in parallel. Each await runs on a separate thread,
    # so this should handle thousands of users simultaneously by using
    # all available cores. The async keyword automatically distributes
    # work across the thread pool."
    
    user = await fetch_user_data(user_id)
    orders = await fetch_user_orders(user_id)
    
    return {"user": user, "orders": orders}

async def main():
    profile = await get_user_profile(42)
    print(profile)

asyncio.run(main())
```

**Your Task:**

Write a detailed code review response that addresses each of the junior's misconceptions. Your response must:

1. **Correct each misconception** specifically:
   - "runs on multiple CPU cores"
   - "processes requests in parallel"
   - "each await runs on a separate thread"
   - "automatically distributes work across the thread pool"

2. **Explain the actual mechanism:** What IS happening when this code runs? How does the event loop actually work?

3. **Identify the bug:** This code has a performance issue related to the sequential awaits. Explain what it is and fix it.

4. **Provide the mental model:** Give them a correct mental model they can use going forward. Use the restaurant analogy or another concrete metaphor.

5. **Connect to prior knowledge:** Reference how this relates to type hints (Lecture 1) and error handling (Lecture 2) — how could this code be improved using those concepts?

---

# PHASE 3: SOLUTION KEY

---

## LEVEL 1 SOLUTION: Predict & Explain

**Correct Answer:**

```
[10:30:00] Starting Binance...
[10:30:02] Finished Binance
[10:30:02] Starting Coinbase...
[10:30:04] Finished Coinbase
[10:30:04] Starting Kraken...
[10:30:06] Finished Kraken

Total time: 6.0 seconds
Fetched 3 prices
```

**The Why:**

The naive model predicts 2 seconds because "async means concurrent." But this code runs **sequentially** — taking 6 seconds total.

**Mental Trace of Execution:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXECUTION TRACE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. main() starts                                               │
│  2. get_all_prices() called                                     │
│                                                                 │
│  3. binance = await fetch_price("Binance")                      │
│     ├─ fetch_price STARTS                                       │
│     ├─ await asyncio.sleep(2) → PAUSES, yields to event loop    │
│     ├─ Event loop: "Any other tasks?" → NO, nothing scheduled   │
│     ├─ Event loop: Waits for sleep to complete                  │
│     ├─ 2 seconds pass                                           │
│     ├─ fetch_price RESUMES, returns result                      │
│     └─ binance = {"exchange": "Binance", ...}                   │
│                                                                 │
│  4. coinbase = await fetch_price("Coinbase")                    │
│     └─ Same process, another 2 seconds                          │
│                                                                 │
│  5. kraken = await fetch_price("Kraken")                        │
│     └─ Same process, another 2 seconds                          │
│                                                                 │
│  TOTAL: 6 seconds                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The Key Insight:**

`await` is a **pause point**, not a parallelization command. When you write:

```python
binance = await fetch_price("Binance")
coinbase = await fetch_price("Coinbase")
```

You're saying: "Wait for Binance to COMPLETE, THEN start Coinbase." The event loop CAN run other tasks during the pause, but **no other tasks were scheduled**.

**The Fix:**

```python
async def get_all_prices() -> list[dict]:
    # Schedule all three BEFORE waiting for any
    return await asyncio.gather(
        fetch_price("Binance"),
        fetch_price("Coinbase"),
        fetch_price("Kraken")
    )
```

Now the event loop has three tasks. When Binance pauses, Coinbase and Kraken can start. Total time: ~2 seconds.

---

## LEVEL 2 SOLUTION: Debug & Fix

**The Flaw:**

Line with the bottleneck:
```python
response = requests.get(url, timeout=5)  # THIS IS BLOCKING!
```

**Explanation:**

`requests.get()` is a **synchronous, blocking** call. When it executes:

1. Python calls into the `requests` library
2. The operating system blocks the thread waiting for network I/O
3. The event loop is **stuck** — it cannot run other coroutines
4. Even though we used `asyncio.gather()`, each request must complete before the next can start

The `await asyncio.sleep(0)` before it does nothing helpful — it yields to the event loop momentarily, but then immediately blocks again with `requests.get()`.

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHAT HAPPENS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Task 1: await sleep(0) → yield → requests.get() → BLOCKED     │
│                                                    ↓            │
│                                          (event loop stuck)     │
│                                                    ↓            │
│  Task 2: (waiting to run, can't start)            ↓            │
│  Task 3: (waiting to run, can't start)            ↓            │
│                                                    ↓            │
│          (5 seconds pass for Task 1's request)    ↓            │
│                                                    ↓            │
│  Task 1: returns, NOW Task 2 can start...         │            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Fixed Code:**

```python
import asyncio
from typing import Optional

# Custom exception hierarchy (Lecture 2 connection)
class ExchangeError(Exception):
    """Base class for exchange-related errors"""
    pass

class ExchangeConnectionError(ExchangeError):
    """Failed to connect to exchange"""
    def __init__(self, exchange: str, reason: str):
        self.exchange = exchange
        self.reason = reason
        super().__init__(f"Failed to connect to {exchange}: {reason}")

class ExchangeTimeoutError(ExchangeError):
    """Request timed out"""
    def __init__(self, exchange: str, timeout: float):
        self.exchange = exchange
        self.timeout = timeout
        super().__init__(f"{exchange} request timed out after {timeout}s")


async def fetch_price(exchange: str, symbol: str) -> dict:
    """Fetch cryptocurrency price from exchange API (non-blocking)"""
    print(f"Fetching {symbol} from {exchange}...")
    
    # Simulate async HTTP call (use httpx in Week 4)
    await asyncio.sleep(1)  # Non-blocking I/O simulation
    
    # Simulate occasional failures
    import random
    if random.random() < 0.1:
        raise ExchangeConnectionError(exchange, "API unavailable")
    
    print(f"Got response from {exchange}")
    return {
        "exchange": exchange, 
        "symbol": symbol, 
        "price": 50000.0 + random.uniform(-100, 100)
    }


async def get_prices_concurrent(
    exchanges: list[str], 
    symbol: str
) -> list[dict | ExchangeError]:
    """
    Fetch from all exchanges concurrently.
    Returns results or exceptions for each exchange.
    """
    tasks = [fetch_price(ex, symbol) for ex in exchanges]
    # return_exceptions=True: Failed tasks return exception objects
    # instead of raising and canceling other tasks
    return await asyncio.gather(*tasks, return_exceptions=True)


async def main():
    import time
    exchanges = ["Binance", "Coinbase", "Kraken", "Gemini", "Bitstamp"]
    start = time.time()
    
    results = await get_prices_concurrent(exchanges, "BTCUSD")
    elapsed = time.time() - start
    
    # Process results, separating successes from failures
    for result in results:
        if isinstance(result, Exception):
            print(f"Error: {result}")
        else:
            print(f"Price from {result['exchange']}: ${result['price']:,.2f}")
    
    print(f"\nFetched in {elapsed:.1f}s (concurrent)")

asyncio.run(main())
```

---

## LEVEL 3 SOLUTION: Design Under Constraints

**Complete Implementation:**

```python
import asyncio
import random
from typing import Optional

PriceResult = dict[str, Optional[float]]

EXCHANGES = [
    "Binance", "Coinbase", "Kraken", "Gemini", "Bitstamp",
    "KuCoin", "Huobi", "OKX", "Bybit", "Gate"
]


async def fetch_single_price(exchange: str) -> float:
    """Simulate fetching price from one exchange."""
    await asyncio.sleep(0.5)
    
    if random.random() < 0.2:
        raise ConnectionError(f"{exchange} API unavailable")
    
    base_price = 50000.0
    variance = random.uniform(-500, 500)
    return base_price + variance


async def safe_fetch(exchange: str) -> tuple[str, Optional[float]]:
    """
    Wrapper that catches exceptions and returns (exchange, result|None).
    This isolates failures so one bad exchange doesn't crash everything.
    """
    try:
        price = await fetch_single_price(exchange)
        return (exchange, price)
    except ConnectionError:
        # Log error in production
        return (exchange, None)
    except Exception as e:
        # Catch unexpected errors too
        print(f"Unexpected error for {exchange}: {e}")
        return (exchange, None)


async def aggregate_prices(exchanges: list[str]) -> PriceResult:
    """
    Fetch from all exchanges concurrently with graceful failure handling.
    """
    # Create all fetch tasks
    tasks = [safe_fetch(exchange) for exchange in exchanges]
    
    # Run ALL tasks concurrently
    # Because safe_fetch catches exceptions, we don't need return_exceptions
    results = await asyncio.gather(*tasks)
    
    # Convert list of tuples to dict
    return {exchange: price for exchange, price in results}


async def main():
    import time
    random.seed(42)  # For reproducible testing
    start = time.time()
    
    results = await aggregate_prices(EXCHANGES)
    elapsed = time.time() - start
    
    print(f"\n{'='*50}")
    print(f"Completed in {elapsed:.2f} seconds")
    print(f"{'='*50}")
    
    successes = {k: v for k, v in results.items() if v is not None}
    failures = {k: v for k, v in results.items() if v is None}
    
    print(f"\nSuccessful: {len(successes)}/{len(results)}")
    for exchange, price in successes.items():
        print(f"  {exchange}: ${price:,.2f}")
    
    if failures:
        print(f"\nFailed: {len(failures)}")
        for exchange in failures:
            print(f"  {exchange}: unavailable")
    
    assert elapsed < 1.0, f"Too slow! {elapsed:.2f}s > 1.0s"
    assert len(results) == 10, f"Missing results: {len(results)}/10"
    print("\n✅ All constraints satisfied!")


if __name__ == "__main__":
    asyncio.run(main())
```

**The Why:**

The key insight is the `safe_fetch` wrapper pattern:

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE PATTERN                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Problem: gather() with 10 tasks. If one fails:                 │
│  ├─ Default: Exception raised, other results lost              │
│  ├─ return_exceptions=True: Get mix of results and exceptions  │
│                                                                 │
│  Solution: Wrapper function that NEVER raises                   │
│                                                                 │
│  async def safe_fetch(exchange):                                │
│      try:                                                       │
│          return (exchange, await fetch_price(exchange))         │
│      except:                                                    │
│          return (exchange, None)  # Failure = None              │
│                                                                 │
│  Now gather() always gets clean results!                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Timing Analysis:**

```
Sequential: 10 × 500ms = 5000ms = 5.0 seconds ❌
Concurrent: max(500ms, 500ms, ...) = 500ms = 0.5 seconds ✅

With some overhead, actual time ≈ 0.51-0.55 seconds
```

---

## LEVEL 4 SOLUTION: Adversarial Counter-Example

**Part A Answers:**

**1. Prediction:**

The three blocks will process **sequentially**, taking approximately **6 seconds** total (not 2 seconds).

**2. Explanation:**

`cpu_intensive_hash` contains **no await points**. It's a pure CPU-bound loop. When it runs:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  gather(block1, block2, block3)                                 │
│     │                                                           │
│     ├─ Starts block1                                            │
│     │   └─ cpu_intensive_hash() runs for 2 seconds              │
│     │      NO AWAIT = NO YIELD = EVENT LOOP BLOCKED             │
│     │                                                           │
│     ├─ block1 finishes, NOW block2 can start                    │
│     │   └─ cpu_intensive_hash() runs for 2 seconds              │
│     │                                                           │
│     └─ block2 finishes, NOW block3 can start                    │
│         └─ cpu_intensive_hash() runs for 2 seconds              │
│                                                                 │
│  Total: 6 seconds                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

The event loop can only switch tasks at `await` points. No `await` = no switching.

**3. Demonstration Code:**

```python
import asyncio
import time

def cpu_intensive_hash(data: bytes) -> str:
    """CPU-intensive work with timing"""
    start = time.time()
    total = 0
    for i in range(10_000_000):
        total += i * len(data)
    elapsed = time.time() - start
    print(f"    Hash computed in {elapsed:.2f}s (blocking!)")
    return hex(total % (2**32))

async def process_block_async(block_id: int) -> dict:
    print(f"  Block {block_id}: Starting at {time.time():.2f}")
    data = f"block_{block_id}".encode()
    hash_result = cpu_intensive_hash(data)
    print(f"  Block {block_id}: Finished at {time.time():.2f}")
    return {"block": block_id, "hash": hash_result}

async def main():
    print("Starting 'concurrent' CPU-bound tasks...")
    start = time.time()
    
    results = await asyncio.gather(
        process_block_async(1),
        process_block_async(2),
        process_block_async(3),
    )
    
    elapsed = time.time() - start
    print(f"\nTotal time: {elapsed:.1f}s")
    print(f"Expected if concurrent: ~{elapsed/3:.1f}s")
    print(f"Actual speedup: {elapsed/3/elapsed:.0%} (none!)")

asyncio.run(main())
```

Output proves sequential execution:
```
Starting 'concurrent' CPU-bound tasks...
  Block 1: Starting at 1705000000.00
    Hash computed in 2.00s (blocking!)
  Block 1: Finished at 1705000002.00
  Block 2: Starting at 1705000002.00
    Hash computed in 2.00s (blocking!)
  Block 2: Finished at 1705000004.00
  Block 3: Starting at 1705000004.00
    Hash computed in 2.00s (blocking!)
  Block 3: Finished at 1705000006.00

Total time: 6.0s
Expected if concurrent: ~2.0s
Actual speedup: 33% (none!)
```

**Part B Answers:**

**4. Two Versions:**

```python
# Synchronous (simple, clear)
def get_user_sync(user_id: int) -> dict:
    response = sync_http_get(f"/users/{user_id}")
    return response.json()

# Asynchronous (more complex, no benefit here)
async def get_user_async(user_id: int) -> dict:
    async with async_http_client() as client:
        response = await client.get(f"/users/{user_id}")
        return response.json()

# To call the async version, you need:
asyncio.run(get_user_async(42))
```

**5. Argument:**

For a **single, sequential request**, sync is better because:

1. **No complexity overhead:** No event loop, no `async/await` keywords, no `asyncio.run()`
2. **No cognitive overhead:** Easier to understand and debug
3. **No performance benefit:** With one request, there's nothing to run concurrently
4. **Simpler stack traces:** Sync exceptions are easier to trace

**Async wins when:**
- You're making multiple independent requests
- You're in an already-async context (like FastAPI)
- You need to handle thousands of concurrent connections

**Rule of thumb:** "If you're only doing one thing at a time, sync is fine."

---

## LEVEL 5 SOLUTION: The Teach-Back

**Code Review Response:**

---

**RE: Code Review for `get_user_profile`**

Hey! Thanks for adding the async implementation. I see you've added some comments explaining your understanding of async — let me help clarify a few things, because there are some common misconceptions here that I had too when I started.

---

**Misconception 1: "runs on multiple CPU cores"**

Actually, Python async runs on a **single thread, single core**. The `asyncio` event loop is a single-threaded cooperative scheduler. It doesn't use multiple cores — that would require `multiprocessing`. 

Think of it this way: one waiter (thread) can serve many tables (tasks) by taking orders, walking to the kitchen, then helping other tables while food cooks. But there's still only one waiter.

---

**Misconception 2: "processes requests in parallel"**

Close, but not quite. Async provides **concurrency**, not **parallelism**:

- **Parallelism:** Two chefs cooking two dishes at the exact same moment (requires multiple CPU cores)
- **Concurrency:** One waiter handling multiple tables by context-switching between them

Your code runs tasks **concurrently** (interleaved), not **in parallel** (simultaneously).

---

**Misconception 3: "each await runs on a separate thread"**

`await` doesn't create threads. Instead, it's a **pause point** — it says: "I'm waiting for I/O. Event loop, run something else while I wait."

When you write `await fetch_user_data(user_id)`:
1. The function runs until it hits its internal `await asyncio.sleep(0.5)`
2. It pauses and returns control to the event loop
3. Event loop checks: "Any other tasks ready to run?" If yes, runs them
4. When the 0.5s passes, the event loop resumes this function

No threads involved. Just cooperative pausing and resuming.

---

**Misconception 4: "automatically distributes work across the thread pool"**

There's no automatic thread pool with `async/await`. Python has `loop.run_in_executor()` for running blocking code in a thread pool, but that's opt-in and separate from normal async.

---

**The Performance Bug:**

Your code has sequential awaits that could be concurrent:

```python
# Current (sequential): Takes 1.0 second
user = await fetch_user_data(user_id)      # 0.5s
orders = await fetch_user_orders(user_id)  # 0.5s (waits for user to finish!)

# Fixed (concurrent): Takes 0.5 seconds
user, orders = await asyncio.gather(
    fetch_user_data(user_id),
    fetch_user_orders(user_id)
)
```

Since `fetch_user_orders` doesn't depend on the result of `fetch_user_data`, they can run concurrently!

---

**The Correct Mental Model:**

Think of async like a **restaurant with one super-efficient waiter**:

| Concept | Restaurant | Python Async |
|---------|------------|--------------|
| Event Loop | Restaurant manager | `asyncio` scheduler |
| Coroutine | An order being processed | `async def` function |
| `await` | "Kitchen is cooking, help another table" | Pause, let other tasks run |
| `gather` | "Take all orders first, deliver as ready" | Run multiple coroutines concurrently |

---

**Improved Code with Type Hints and Error Handling (Lecture 1 & 2):**

```python
import asyncio
from typing import TypedDict

# Type hints (Lecture 1)
class User(TypedDict):
    id: int
    name: str

class Order(TypedDict):
    order_id: int
    user_id: int

class UserProfile(TypedDict):
    user: User
    orders: list[Order]

# Custom exception (Lecture 2)
class UserNotFoundError(Exception):
    def __init__(self, user_id: int):
        self.user_id = user_id
        super().__init__(f"User {user_id} not found")


async def fetch_user_data(user_id: int) -> User:
    await asyncio.sleep(0.5)
    if user_id < 0:
        raise UserNotFoundError(user_id)
    return {"id": user_id, "name": f"User_{user_id}"}


async def fetch_user_orders(user_id: int) -> list[Order]:
    await asyncio.sleep(0.5)
    return [{"order_id": i, "user_id": user_id} for i in range(3)]


async def get_user_profile(user_id: int) -> UserProfile:
    """
    Fetch user profile with concurrent database queries.
    
    Note: This uses async for concurrent I/O, running on a single thread.
    The event loop switches between tasks at await points.
    """
    # Concurrent fetch — both queries run "at the same time"
    user, orders = await asyncio.gather(
        fetch_user_data(user_id),
        fetch_user_orders(user_id)
    )
    
    return {"user": user, "orders": orders}
```

---

**Summary Table:**

| What they thought | What's actually true |
|-------------------|---------------------|
| Multiple cores | Single thread, single core |
| Parallel execution | Concurrent (interleaved) execution |
| `await` = new thread | `await` = pause point |
| Automatic thread pool | Explicit opt-in only |

Hope this helps clarify things! Async is powerful for I/O-bound work, but understanding the single-threaded nature is key to using it correctly. Happy to chat more if you have questions!

---