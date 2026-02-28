# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BRIDGE, DON'T START OVER                                       │
│  ────────────────────────                                       │
│  Students have been BUILDING production systems for 16 weeks.   │
│  They already use data structures and algorithms daily —        │
│  they just don't call them by their interview names.            │
│  Show them what they already know before teaching anything new. │
│                                                                 │
│  FEEL THE DIFFERENCE                                            │
│  ───────────────────                                            │
│  Like our sync vs async demo in Week 1, we'll TIME two          │
│  approaches live. Make O(n²) vs O(n) visceral, not theoretical. │
│                                                                 │
│  STRATEGY OVER ENCYCLOPEDIA                                     │
│  ─────────────────────────                                      │
│  This is NOT a DSA course crammed into one lecture.             │
│  It's a guided framework for backend interviews specifically.   │
│  We teach the 20% of patterns that cover 80% of questions.     │
│                                                                 │
│  OWN CODE FIRST, LEETCODE SECOND                                │
│  ────────────────────────────────                               │
│  Analyze complexity of code THEY wrote — Task Manager queries,  │
│  caching logic, Celery tasks — then generalize to abstract      │
│  interview problems. The abstract becomes concrete.             │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  EXPLAIN ANALYZE (Week 7)  → You already do complexity analysis │
│  Redis data types (Week 10) → Interview data structures         │
│  Celery queues (Week 11)    → Queue pattern in DSA              │
│  ADRs / Capstone (Week 14)  → Behavioral story source          │
│  System design (Week 16 L1-L2) → System design round prep      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                DSA & INTERVIEW PREP STRATEGY                    │
│                     (3.5-4 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE INTERVIEW LANDSCAPE (30 min)                       │
│  ├─ 1.1 The Driving Test (Analogy Introduction)                 │
│  ├─ 1.2 You Already Know More Than You Think                    │
│  ├─ 1.3 Backend Interview Structure                             │
│  └─ 1.4 What Interviewers Actually Evaluate                     │
│                                                                 │
│  PART 2: COMPLEXITY ANALYSIS (55 min)                           │
│  ├─ 2.1 Big O: The Speedometer of Code                          │
│  ├─ 2.2 Time Complexity Fundamentals                            │
│  ├─ 2.3 The Complexity Hierarchy (with YOUR Code)               │
│  ├─ 2.4 Space Complexity                                        │
│  └─ 2.5 Analyzing Code You've Already Written                   │
│                                                                 │
│  PART 3: BACKEND DSA PATTERNS (60 min)                          │
│  ├─ 3.1 The Pattern Recognition Mindset                         │
│  ├─ 3.2 Hash Maps — Your Best Friend                            │
│  ├─ 3.3 Stacks and Queues — Order Matters                       │
│  ├─ 3.4 Trees — Hierarchical Thinking                           │
│  ├─ 3.5 Graphs — Everything is Connected                        │
│  └─ 3.6 The Backend Pattern Cheat Sheet                         │
│                                                                 │
│  PART 4: SQL INTERVIEW MASTERY (45 min)                         │
│  ├─ 4.1 Why Backend Interviews Test SQL Hard                    │
│  ├─ 4.2 Window Functions (The Interview Differentiator)         │
│  ├─ 4.3 Common SQL Interview Patterns                           │
│  └─ 4.4 The SQL Problem-Solving Framework                       │
│                                                                 │
│  PART 5: BEHAVIORAL & PORTFOLIO (30 min)                        │
│  ├─ 5.1 The STAR Method (Commentary Driving)                    │
│  ├─ 5.2 Mining Your Capstone for Stories                        │
│  ├─ 5.3 The "Walk Me Through Your Project" Framework            │
│  └─ 5.4 Common Behavioral Mistakes                              │
│                                                                 │
│  PART 6: THE PRACTICE STRATEGY (15 min)                         │
│  ├─ 6.1 The 80/20 Rule for Backend DSA                          │
│  ├─ 6.2 The Interview Day Playbook                              │
│  └─ 6.3 Continued Growth After This Course                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE INTERVIEW LANDSCAPE

## 1.1 The Driving Test

**Start with a reframe. Reduce the anxiety.**

> "You've been driving for 16 weeks. You built a multi-tenant SaaS backend. You've deployed to the cloud. You handle WebSockets, background jobs, caching, auth — production-grade work. You are a competent driver. But now... you need to pass the driving test."

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE DRIVING TEST ANALOGY                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DRIVING EXPERIENCE (16 weeks of building)                      │
│  ─────────────────────────────────────────                      │
│  You've driven across the country.                              │
│  You navigated storms, detours, breakdowns.                     │
│  You're a confident, capable driver.                            │
│                                                                 │
│  THE DRIVING TEST (Technical Interview)                         │
│  ─────────────────────────────────────                          │
│  Doesn't ask you to drive across the country.                   │
│  Asks you to:                                                   │
│  ├─ Parallel park       → Solve a specific algorithm problem    │
│  ├─ Identify road signs → Explain Big O complexity              │
│  ├─ Commentary driving  → Think out loud while solving          │
│  ├─ Hazard perception   → Handle edge cases                     │
│  ├─ Driving history     → Behavioral / experience questions     │
│  └─ Route planning      → System design                        │
│                                                                 │
│  THE GAP:                                                       │
│  Being a great driver ≠ automatically passing the test.         │
│  But the test is LEARNABLE, and your experience gives you       │
│  a massive advantage over someone who only studied theory.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight:**

> "The interview is NOT a measure of your ability to build software. It's a specific FORMAT with specific RULES. You already have the skills. This lecture teaches you the format."

---

## 1.2 You Already Know More Than You Think

**Before we teach anything new, let's prove what you already know.**

```
┌─────────────────────────────────────────────────────────────────┐
│           WHAT YOU BUILT  →  WHAT INTERVIEWERS TEST             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOUR EXPERIENCE                   DSA / INTERVIEW CONCEPT      │
│  ────────────────                  ──────────────────────       │
│  Redis cache key lookup            Hash map O(1) access         │
│  Celery task queue                 Queue data structure          │
│  FastAPI middleware chain           Stack (LIFO processing)      │
│  PostgreSQL B-tree indexes          Balanced tree → O(log n)     │
│  Alembic migration ordering         Topological sort (graphs)    │
│  EXPLAIN: Seq Scan → Index Scan     O(n) → O(log n) tradeoff    │
│  N+1 query → joinedload            Nested loop optimization     │
│  Cache-aside pattern               Memoization / lookup table   │
│  Rate limiter (token bucket)       Sliding window algorithm     │
│  Cursor pagination                 Pointer-based traversal      │
│  WebSocket connection manager      Graph: adjacency list        │
│  Docker Compose service deps       Directed Acyclic Graph       │
│                                                                 │
│  You've been DOING data structures and algorithms               │
│  for 16 weeks. You just didn't call them that.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "When you added an index to your PostgreSQL table in Week 7 and watched the EXPLAIN output change from Seq Scan to Index Scan — what did that feel like? That was O(n) → O(log n). You already think in complexity. Today we give it a formal name."

---

## 1.3 Backend Interview Structure

**Know what you're walking into:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TYPICAL BACKEND INTERVIEW PIPELINE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ROUND 1: RECRUITER SCREEN (30 min)                             │
│  ├─ "Tell me about yourself"                                    │
│  ├─ Role fit, salary expectations, timeline                     │
│  └─ Tip: Have a 2-minute pitch about your capstone ready        │
│                                                                 │
│  ROUND 2: TECHNICAL PHONE SCREEN (45-60 min)                    │
│  ├─ 1-2 coding problems (LeetCode Easy/Medium)                  │
│  ├─ Usually in a shared editor (CoderPad, HackerRank)           │
│  └─ Often includes a SQL question for backend roles             │
│                                                                 │
│  ROUND 3: ON-SITE / VIRTUAL LOOP (4-5 hours)                   │
│  ├─ DSA Round (45 min × 1-2): Algorithm problems                │
│  ├─ System Design (45-60 min): Design a backend system          │
│  ├─ Backend Deep Dive (45 min): API design, DB schema, etc.     │
│  ├─ Behavioral (30-45 min): Past experience, teamwork           │
│  └─ Sometimes: Live coding / take-home project                  │
│                                                                 │
│  ROUND 4: HIRING MANAGER (30 min)                               │
│  ├─ Team fit, growth, career goals                              │
│  └─ Your chance to evaluate THEM                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Backend interviews are different from generic SWE interviews:**

```
┌─────────────────────────────────────────────────────────────────┐
│          GENERIC SWE INTERVIEW VS BACKEND INTERVIEW             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GENERIC SWE                       BACKEND-SPECIFIC             │
│  ───────────                       ────────────────             │
│  Heavy DSA (trees, DP,             Moderate DSA (hash maps,     │
│    graph theory)                     queues, common patterns)   │
│                                                                 │
│  Frontend/fullstack                SQL is almost GUARANTEED     │
│    questions mixed in                                           │
│                                                                 │
│  System design is general          System design focuses on     │
│                                      APIs, databases, scaling   │
│                                                                 │
│  Behavioral is generic             Deep dives on YOUR backend   │
│                                      decisions (schema design,  │
│                                      caching strategy, etc.)    │
│                                                                 │
│  THE GOOD NEWS FOR YOU:                                         │
│  Backend interviews test EXACTLY what you've been building.     │
│  The DSA portion is usually the SMALLEST gap to close.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 What Interviewers Actually Evaluate

**This changes everything about how you practice.**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT INTERVIEWERS ACTUALLY SCORE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  It's NOT just "did you solve the problem."                     │
│                                                                 │
│  ┌──────────────────────────────┬──────────────────┐            │
│  │      WHAT THEY EVALUATE     │  WEIGHT (approx)  │            │
│  ├──────────────────────────────┼──────────────────┤            │
│  │ Problem-solving PROCESS     │      35%          │            │
│  │ (How you break it down)     │                   │            │
│  ├──────────────────────────────┼──────────────────┤            │
│  │ Communication               │      25%          │            │
│  │ (Thinking out loud,         │                   │            │
│  │  clarifying questions)      │                   │            │
│  ├──────────────────────────────┼──────────────────┤            │
│  │ Code quality                │      20%          │            │
│  │ (Clean, readable, handles   │                   │            │
│  │  edge cases)                │                   │            │
│  ├──────────────────────────────┼──────────────────┤            │
│  │ Correctness                 │      15%          │            │
│  │ (Does it actually work?)    │                   │            │
│  ├──────────────────────────────┼──────────────────┤            │
│  │ Optimization                │       5%          │            │
│  │ (Optimal Big O)             │                   │            │
│  └──────────────────────────────┴──────────────────┘            │
│                                                                 │
│  SURPRISE: Getting the optimal solution is only 5%.             │
│  A candidate who solves brute force clearly, communicates       │
│  well, and discusses tradeoffs often BEATS the candidate        │
│  who silently writes optimal code.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The interview communication protocol:**

```
┌─────────────────────────────────────────────────────────────────┐
│             THE 5-STEP INTERVIEW PROTOCOL                       │
│             (Use this for EVERY problem)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: CLARIFY (2-3 min)                                      │
│  ├─ Repeat the problem in your own words                        │
│  ├─ Ask about edge cases: empty input? duplicates? negatives?   │
│  ├─ Ask about constraints: input size? time/space requirements? │
│  └─ "Just to confirm: I receive X and need to return Y?"        │
│                                                                 │
│  STEP 2: EXAMPLES (2-3 min)                                     │
│  ├─ Work through 1-2 examples by hand                           │
│  ├─ Include a normal case AND an edge case                      │
│  └─ "Let me trace through an example to make sure I understand" │
│                                                                 │
│  STEP 3: APPROACH (3-5 min)                                     │
│  ├─ State your brute force approach and its complexity           │
│  ├─ Explain WHY it's suboptimal                                 │
│  ├─ Describe your optimized approach (if you see one)           │
│  └─ "I'll start with brute force, then optimize — sound good?"  │
│                                                                 │
│  STEP 4: CODE (15-20 min)                                       │
│  ├─ Write clean code, not pseudocode                            │
│  ├─ Talk while you code: "Now I'm iterating to find..."         │
│  ├─ Use meaningful variable names (not i, j, k everywhere)      │
│  └─ Handle edge cases explicitly                                │
│                                                                 │
│  STEP 5: VERIFY (3-5 min)                                       │
│  ├─ Trace through your code with your example                   │
│  ├─ Test edge cases mentally                                    │
│  ├─ State the final time and space complexity                   │
│  └─ "This runs in O(n) time and O(n) space because..."          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This 5-step protocol is like commentary driving — you narrate what you're doing and why. Interviewers can't read your mind. If you think silently for 5 minutes, they don't know if you're having a breakthrough or panicking. TALK."

---

# PART 2: COMPLEXITY ANALYSIS

## 2.1 Big O: The Speedometer of Code

**Start with the question that motivates everything:**

> "How fast is your code?"

The answer is NEVER a number of seconds. Seconds depend on the machine, the input, the weather. The real question is: **how does performance SCALE as input grows?**

```
┌─────────────────────────────────────────────────────────────────┐
│                     WHAT IS BIG O?                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Big O describes HOW YOUR CODE'S PERFORMANCE SCALES             │
│  as the input size (n) grows.                                   │
│                                                                 │
│  It answers: "If I double the input, what happens to the time?" │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ Big O    │ Name          │ Double input → time...   │        │
│  ├──────────┼───────────────┼──────────────────────────┤        │
│  │ O(1)     │ Constant      │ Stays the same           │        │
│  │ O(log n) │ Logarithmic   │ Barely increases          │        │
│  │ O(n)     │ Linear        │ Doubles                  │        │
│  │ O(n log n)│ Linearithmic │ Slightly more than doubles│        │
│  │ O(n²)    │ Quadratic     │ Quadruples (4x)          │        │
│  │ O(2ⁿ)   │ Exponential   │ Squares itself           │        │
│  └──────────┴───────────────┴──────────────────────────┘        │
│                                                                 │
│  THE RULES:                                                     │
│  1. Drop constants:     O(3n) → O(n)                            │
│  2. Drop lower terms:   O(n² + n) → O(n²)                      │
│  3. We care about WORST CASE (usually)                          │
│  4. We care about the SHAPE of growth, not exact numbers        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why drop constants?**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHY DROP CONSTANTS?                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "But 3n is three times slower than n!"                         │
│                                                                 │
│  Yes — at small scale. But watch what happens at scale:         │
│                                                                 │
│  n = 1,000,000                                                  │
│  ├─ O(3n)  = 3,000,000 operations                               │
│  ├─ O(n²)  = 1,000,000,000,000 operations                      │
│  │                                                              │
│  │  The constant 3 is a rounding error compared to              │
│  │  the SHAPE difference between n and n².                      │
│  │                                                              │
│  │  This is like worrying about the color of your car           │
│  │  when choosing between a car and a bicycle for               │
│  │  a cross-country trip. The vehicle TYPE matters.             │
│  │  The color doesn't.                                          │
│                                                                 │
│  Big O captures the VEHICLE TYPE of your algorithm.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 Time Complexity Fundamentals

**Let's make this concrete. Every complexity level, with code you can read.**

### O(1) — Constant Time

```python
# The operation takes the same time regardless of input size.
# You've been doing this all course long.

def get_by_key(data: dict[str, int], key: str) -> int | None:
    return data.get(key)  # O(1) — dict lookup is constant time

# Doesn't matter if the dict has 10 items or 10 million.
# Roughly the same speed.
```

```
WHERE YOU'VE SEEN THIS:
├─ Redis GET/SET                      (Week 10)
├─ dict lookups in your cache layer   (Week 10)
├─ Accessing a list element by index  tasks[0]
└─ Your hash-based token validation   (Week 9)
```

### O(log n) — Logarithmic Time

```python
# Each step HALVES the remaining work.
# Classic example: binary search.

def binary_search(sorted_list: list[int], target: int) -> int | None:
    """Find target in a sorted list. Return index or None."""
    left, right = 0, len(sorted_list) - 1

    while left <= right:                  # Loop runs log₂(n) times
        mid = (left + right) // 2
        if sorted_list[mid] == target:
            return mid
        elif sorted_list[mid] < target:
            left = mid + 1                # Throw away left half
        else:
            right = mid - 1               # Throw away right half

    return None
```

```
WHY log n?
─────────
n = 1,000,000 items
After step 1:  500,000 remain
After step 2:  250,000 remain
After step 3:  125,000 remain
...
After step 20: 1 item remains    ← Found it in ~20 steps!

log₂(1,000,000) ≈ 20

A million items, 20 steps. A billion items? Only 30 steps.
```

```
WHERE YOU'VE SEEN THIS:
├─ PostgreSQL B-tree index lookup     (Week 7)
│  └─ EXPLAIN showed Index Scan — that's O(log n)!
├─ Binary search debugging strategy   (Week 2)
│  └─ "Cut the problem space in half each time"
└─ Sorted set operations in Redis     (Week 10)
```

### O(n) — Linear Time

```python
# You touch every element once. Double the input, double the time.

def find_max(numbers: list[int]) -> int:
    """Must look at every element to find the maximum."""
    maximum = numbers[0]
    for num in numbers:       # O(n) — one pass through the list
        if num > maximum:
            maximum = num
    return maximum
```

```
WHERE YOU'VE SEEN THIS:
├─ Iterating through query results    (all DB-backed endpoints)
├─ Serializing a list of Pydantic models to JSON
├─ PostgreSQL Seq Scan (no index)     (Week 7)
└─ Your load test processing results  (Week 12)
```

### O(n log n) — Linearithmic Time

```python
# Sorting. Almost every language's built-in sort is O(n log n).

def get_sorted_tasks(tasks: list[dict]) -> list[dict]:
    return sorted(tasks, key=lambda t: t["created_at"])  # O(n log n)
```

```
WHERE YOU'VE SEEN THIS:
├─ ORDER BY in your SQL queries       (PostgreSQL uses merge/quick sort)
├─ Python's sorted() and .sort()
└─ Any time you sorted API responses before pagination
```

### O(n²) — Quadratic Time

```python
# For every element, you look at every OTHER element.
# Double the input → 4x the time. THIS IS THE DANGER ZONE.

def find_duplicates_brute(items: list[str]) -> list[str]:
    """Check every pair — painfully slow."""
    duplicates = []
    for i in range(len(items)):             # O(n)
        for j in range(i + 1, len(items)):  #   × O(n)
            if items[i] == items[j]:        # = O(n²)
                duplicates.append(items[i])
    return duplicates
```

```
WHERE YOU'VE SEEN THIS (and fixed it):
├─ N+1 query problem  (Week 6)
│  └─ For each task, query tags separately = n queries × m rows each
│  └─ Fixed with joinedload/selectinload → one query
├─ Nested loop in your early Task Manager code
└─ Any time EXPLAIN showed Nested Loop with Seq Scan
```

### O(2ⁿ) — Exponential Time

```python
# Every element doubles the work. This explodes fast.
# Appears in: brute-force subset generation, naive recursion.

def all_subsets(items: list[str]) -> list[list[str]]:
    """Generate every possible subset. DON'T do this for large inputs."""
    if not items:
        return [[]]
    first = items[0]
    rest_subsets = all_subsets(items[1:])  # Recursive: 2 branches each level
    return rest_subsets + [[first] + s for s in rest_subsets]

# n=10:  1,024 subsets
# n=20:  1,048,576 subsets
# n=30:  1,073,741,824 subsets — good luck.
```

```
YOU PROBABLY HAVEN'T WRITTEN THIS:
And that's fine. You'll rarely encounter exponential algorithms
in backend work. Recognize it so you can AVOID it.
```

---

## 2.3 The Complexity Hierarchy (with YOUR Code)

**Now, let's FEEL the difference. Run this demo.**

```python
# demo_complexity.py — Run this with students watching
import time

# --- O(n) approach: Hash set ---
def has_duplicate_set(items: list[int]) -> bool:
    seen: set[int] = set()
    for item in items:
        if item in seen:          # set lookup = O(1)
            return True
        seen.add(item)
    return False

# --- O(n²) approach: Nested loop ---
def has_duplicate_brute(items: list[int]) -> bool:
    for i in range(len(items)):
        for j in range(i + 1, len(items)):
            if items[i] == items[j]:
                return True
    return False

# --- Time them at increasing scale ---
print(f"{'n':>10} | {'O(n²) brute':>14} | {'O(n) hash set':>14} | {'Speedup':>10}")
print("-" * 58)

for n in [1_000, 5_000, 10_000, 30_000, 50_000]:
    items = list(range(n))  # No duplicates — worst case, must check all

    start = time.time()
    has_duplicate_brute(items)
    brute_time = time.time() - start

    start = time.time()
    has_duplicate_set(items)
    set_time = time.time() - start

    speedup = brute_time / set_time if set_time > 0 else float('inf')
    print(f"{n:>10} | {brute_time:>12.4f}s | {set_time:>12.6f}s | {speedup:>8.0f}x")
```

**Run it. Watch the clock. Expect something like this:**

```
         n |   O(n²) brute |  O(n) hash set |    Speedup
----------------------------------------------------------
     1,000 |       0.0300s |     0.000050s  |      600x
     5,000 |       0.7500s |     0.000250s  |     3000x
    10,000 |       3.0000s |     0.000500s  |     6000x
    30,000 |      27.0000s |     0.001500s  |    18000x
    50,000 |      75.0000s |     0.002500s  |    30000x
```

**Ask the class:**

> "At 50,000 items, the brute force takes over a minute. The hash set takes 2 milliseconds. Same result. Same input. Different algorithm. THIS is why Big O matters — and this is exactly what happened when you added database indexes in Week 7."

```
┌─────────────────────────────────────────────────────────────────┐
│                GROWTH RATES VISUALIZED                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Operations                                                     │
│  │                                                              │
│  │                                              ╱ O(2ⁿ)        │
│  │                                           ╱╱╱               │
│  │                                        ╱╱╱                  │
│  │                                     ╱╱╱                     │
│  │                                                              │
│  │                                    ╱ O(n²)                   │
│  │                                 ╱╱╱                          │
│  │                              ╱╱╱                             │
│  │                           ╱╱╱                                │
│  │                        ╱╱╱                                   │
│  │                     ╱╱╱             ╱╱╱ O(n log n)           │
│  │                  ╱╱╱          ╱╱╱╱╱╱                         │
│  │               ╱╱╱      ╱╱╱╱╱╱                               │
│  │           ╱╱╱╱╱╱╱╱╱╱╱╱╱    ╱╱╱╱ O(n)                       │
│  │      ╱╱╱╱╱       ╱╱╱╱╱╱╱╱╱                                 │
│  │ ╱╱╱╱╱  ╱╱╱╱╱╱╱╱╱╱                                          │
│  ├─────────────────────────────── O(log n)                      │
│  ├═══════════════════════════════ O(1)                           │
│  └──────────────────────────────────────────── n (input size)   │
│                                                                 │
│  KEY TAKEAWAY:                                                  │
│  The SHAPE matters more than the starting point.                │
│  O(n²) ALWAYS overtakes O(n) eventually, no matter              │
│  how fast your hardware is.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Space Complexity

**Same idea, applied to MEMORY instead of time.**

> "Time complexity: how much LONGER does it take as input grows?
> Space complexity: how much MORE MEMORY does it need as input grows?"

```python
# O(1) space — uses fixed amount of memory regardless of input
def find_max(numbers: list[int]) -> int:
    maximum = numbers[0]       # One variable, constant space
    for num in numbers:
        if num > maximum:
            maximum = num
    return maximum

# O(n) space — creates a new collection proportional to input
def get_unique(items: list[str]) -> list[str]:
    seen: set[str] = set()     # Could grow to n items
    result: list[str] = []     # Could grow to n items
    for item in items:
        if item not in seen:
            seen.add(item)
            result.append(item)
    return result
```

```
┌─────────────────────────────────────────────────────────────────┐
│             THE TIME-SPACE TRADEOFF                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You often TRADE space for time. Use more memory → run faster.  │
│                                                                 │
│  EXAMPLE: Finding duplicates                                    │
│                                                                 │
│  Approach 1: No extra memory                                    │
│  ├─ Time: O(n²) — compare every pair                            │
│  └─ Space: O(1) — no extra storage                              │
│                                                                 │
│  Approach 2: Use a hash set                                     │
│  ├─ Time: O(n) — one pass, set lookup is O(1)                   │
│  └─ Space: O(n) — the set could hold n items                    │
│                                                                 │
│  YOU ALREADY KNOW THIS TRADEOFF:                                │
│  ├─ Redis cache (Week 10): use memory → faster responses        │
│  ├─ Database indexes (Week 7): use disk space → faster queries  │
│  └─ Denormalization (Week 5): duplicate data → fewer joins      │
│                                                                 │
│  In interviews, the optimized approach usually trades           │
│  O(n) space for a better time complexity. State this tradeoff   │
│  explicitly: "I can improve from O(n²) to O(n) time by         │
│  using O(n) additional space for a hash map."                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.5 Analyzing Code You've Already Written

**Let's apply complexity analysis to patterns from YOUR projects.**

### Pattern: Your Cache-Aside Logic (Week 10)

```python
async def get_task(task_id: int, redis: Redis, db: AsyncSession) -> Task:
    # Step 1: Check cache — O(1) time
    cached = await redis.get(f"task:{task_id}")   # O(1) Redis lookup
    if cached:
        return Task.model_validate_json(cached)   # O(1) deserialization

    # Step 2: Query database — O(log n) with index, O(n) without
    result = await db.execute(
        select(TaskModel).where(TaskModel.id == task_id)  # Indexed PK → O(log n)
    )
    task = result.scalar_one()

    # Step 3: Populate cache — O(1) Redis SET
    await redis.set(
        f"task:{task_id}",
        task.model_dump_json(),
        ex=3600
    )

    return Task.model_validate(task)

# TOTAL COMPLEXITY:
# Best case (cache hit):   O(1) time, O(1) space
# Worst case (cache miss): O(log n) time (indexed DB lookup), O(1) space
# This is EXACTLY the pattern behind memoization in DSA problems!
```

### Pattern: Your List Endpoint with Filtering (Week 3-4, later enhanced)

```python
# EARLY VERSION (in-memory, Week 3):
def get_tasks_by_status_and_tag(
    tasks: list[Task],
    status: str,
    tag: str
) -> list[Task]:
    result = []
    for task in tasks:                     # O(n)
        if task.status == status:          #   O(1) comparison
            for t in task.tags:            #   O(m) per task — inner loop!
                if t == tag:
                    result.append(task)
    return result

# COMPLEXITY: O(n × m) where n = tasks, m = avg tags per task
# When you moved to PostgreSQL with proper indexes and JOINs,
# you handed this work to the database's optimized query planner.
```

### Pattern: Your N+1 Problem Fix (Week 6)

```python
# BEFORE (N+1 — you felt this pain):
async def get_all_tasks_bad(db: AsyncSession) -> list[Task]:
    result = await db.execute(select(TaskModel))       # 1 query
    tasks = result.scalars().all()
    for task in tasks:                                  # n iterations
        _ = task.tags  # Each access triggers ANOTHER query!  # n queries
    return tasks
# TOTAL QUERIES: 1 + n = O(n) database round trips
# Each round trip is I/O-bound. n round trips is devastating.

# AFTER (joinedload — one query):
async def get_all_tasks_good(db: AsyncSession) -> list[Task]:
    result = await db.execute(
        select(TaskModel).options(joinedload(TaskModel.tags))  # 1 query
    )
    tasks = result.unique().scalars().all()
    return tasks
# TOTAL QUERIES: 1 = O(1) database round trips
# You optimized from O(n) I/O operations to O(1).
# In DSA terms: you eliminated the nested "query" loop.
```

**The insight:**

> "You've been doing complexity analysis for months. Every time you looked at EXPLAIN output and decided to add an index, every time you fixed an N+1 query, every time you cached a response — you were making the same decisions that interview problems test. The difference is that interviews ask you to do it with arrays and hash maps instead of databases and Redis."

---

# PART 3: BACKEND DSA PATTERNS

## 3.1 The Pattern Recognition Mindset

**Interviews aren't about memorizing 500 solutions. They're about recognizing PATTERNS.**

```
┌─────────────────────────────────────────────────────────────────┐
│             THE PATTERN RECOGNITION FRAMEWORK                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  When you see a problem, ask these questions IN ORDER:          │
│                                                                 │
│  1. "Can I use a HASH MAP to avoid a nested loop?"              │
│     └─ If you're searching/matching/counting → probably yes     │
│     └─ This solves ~40% of interview problems                   │
│                                                                 │
│  2. "Does ORDER matter? Do I need FIFO or LIFO?"                │
│     └─ FIFO (first-in-first-out) → Queue                       │
│     └─ LIFO (last-in-first-out) → Stack                        │
│                                                                 │
│  3. "Is the data HIERARCHICAL or SORTED?"                       │
│     └─ Hierarchical → Tree traversal                            │
│     └─ Sorted → Binary search                                  │
│                                                                 │
│  4. "Are there RELATIONSHIPS or CONNECTIONS between items?"     │
│     └─ Relationships → Graph (BFS/DFS)                          │
│                                                                 │
│  5. "Am I looking for a CONTIGUOUS RANGE in a sequence?"        │
│     └─ Subarray/substring → Sliding window                     │
│                                                                 │
│  6. "Do I need to try COMBINATIONS or PERMUTATIONS?"            │
│     └─ Yes → Backtracking (less common in backend interviews)   │
│                                                                 │
│  Start with question 1. You'll be surprised how often           │
│  the answer is "yes, use a hash map."                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Hash Maps — Your Best Friend

**You use `dict` constantly. Now understand WHY it's powerful, and how to wield it in interviews.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     HASH MAP ESSENTIALS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Python dict / set:                                             │
│  ├─ Lookup:  O(1) average                                       │
│  ├─ Insert:  O(1) average                                       │
│  ├─ Delete:  O(1) average                                       │
│  └─ Space:   O(n)                                               │
│                                                                 │
│  WHY O(1)? The key is hashed to a memory address.               │
│  Instead of searching through everything, you jump              │
│  directly to where the value lives. Like a library              │
│  catalog number vs. searching every shelf.                      │
│                                                                 │
│  INTERVIEW SUPERPOWER:                                          │
│  Whenever you see a nested loop where the inner loop            │
│  is SEARCHING for something, replace the search with            │
│  a hash map lookup.                                             │
│                                                                 │
│  O(n) × O(n) search = O(n²)                                    │
│  O(n) × O(1) lookup = O(n)  ← Hash map wins                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The Classic: Two Sum (the most-asked interview question in history)**

> "Given a list of numbers and a target sum, find two numbers that add up to the target. Return their indices."

```python
# STEP 1 (in interview): State brute force first.
def two_sum_brute(nums: list[int], target: int) -> tuple[int, int] | None:
    """Check every pair. O(n²) time, O(1) space."""
    for i in range(len(nums)):
        for j in range(i + 1, len(nums)):
            if nums[i] + nums[j] == target:
                return (i, j)
    return None

# "This works but it's O(n²). For each number, I'm scanning
# the rest of the list to find its complement.
# I can do better by remembering what I've already seen."

# STEP 2: Optimize with a hash map.
def two_sum(nums: list[int], target: int) -> tuple[int, int] | None:
    """One pass with a hash map. O(n) time, O(n) space."""
    seen: dict[int, int] = {}  # value → index

    for i, num in enumerate(nums):
        complement = target - num

        if complement in seen:      # O(1) lookup!
            return (seen[complement], i)

        seen[num] = i               # Remember this number

    return None
```

**Trace through an example (do this live in the interview):**

```
nums = [2, 7, 11, 15], target = 9

Step 0: i=0, num=2, complement=7, seen={}
        7 not in seen → store {2: 0}

Step 1: i=1, num=7, complement=2, seen={2: 0}
        2 IS in seen! → return (0, 1) ✅

Answer: indices (0, 1) — nums[0] + nums[1] = 2 + 7 = 9
```

**Ask the class:**

> "Recognize the pattern? This is your cache-aside logic from Week 10. Check the fast lookup first (hash map / Redis). If not there, do the work and store it for later. Same principle, different context."

**Hash Map Pattern: Frequency Counting**

```python
# "Which element appears most frequently?"
# Backend connection: this is how rate limiting works —
# counting requests per user/IP.

from collections import Counter

def most_frequent(items: list[str]) -> str:
    """Find the most common element. O(n) time, O(n) space."""
    counts: dict[str, int] = {}

    for item in items:
        counts[item] = counts.get(item, 0) + 1   # O(1) per operation

    max_item = ""
    max_count = 0
    for item, count in counts.items():
        if count > max_count:
            max_count = count
            max_item = item

    return max_item

# Or simply: Counter(items).most_common(1)[0][0]
# But in interviews, showing the manual approach demonstrates understanding.
```

**Hash Map Pattern: Grouping / Anagram Detection**

```python
# "Group strings that are anagrams of each other."
# Backend connection: this is data normalization —
# different inputs mapped to the same canonical form.

def group_anagrams(words: list[str]) -> list[list[str]]:
    """Group words by their sorted character signature. O(n × k log k)."""
    groups: dict[str, list[str]] = {}

    for word in words:
        key = "".join(sorted(word))  # Canonical form: "eat" → "aet"
        if key not in groups:
            groups[key] = []
        groups[key].append(word)

    return list(groups.values())

# group_anagrams(["eat", "tea", "tan", "ate", "nat", "bat"])
# → [["eat", "tea", "ate"], ["tan", "nat"], ["bat"]]
```

```
┌─────────────────────────────────────────────────────────────────┐
│              HASH MAP INTERVIEW SIGNALS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  When you hear...              Think hash map:                  │
│  ───────────────               ───────────────                  │
│  "Find a pair that..."         → Store complement, look up      │
│  "Count occurrences..."        → Frequency counter              │
│  "Find duplicates..."          → Seen set                       │
│  "Group by some property..."   → Key = property, value = group  │
│  "First non-repeating..."      → Frequency map, then scan       │
│  "Check if exists..."          → Set membership O(1)            │
│  "Map one thing to another..." → Direct key-value mapping       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Stacks and Queues — Order Matters

**Stacks: Last-In, First-Out (LIFO)**

```
┌─────────────────────────────────────────────────────────────────┐
│                        STACK (LIFO)                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Think: a stack of plates. You add to the top, remove from top. │
│                                                                 │
│    push("A") → push("B") → push("C") → pop() returns "C"      │
│                                                                 │
│     ┌───┐         ┌───┐                                        │
│     │ C │ ← top   │ B │ ← top (after pop)                      │
│     ├───┤         ├───┤                                         │
│     │ B │         │ A │                                         │
│     ├───┤         └───┘                                         │
│     │ A │                                                       │
│     └───┘                                                       │
│                                                                 │
│  In Python: use a list                                          │
│  ├─ push → list.append(item)    O(1)                            │
│  ├─ pop  → list.pop()           O(1)                            │
│  └─ peek → list[-1]             O(1)                            │
│                                                                 │
│  WHERE YOU'VE SEEN THIS:                                        │
│  ├─ FastAPI middleware chain: each middleware wraps the next.    │
│  │  The LAST middleware added is the FIRST to process.          │
│  ├─ Python call stack: function calls stack up, return in       │
│  │  reverse order. Stack traces (Week 2) show this literally.  │
│  └─ Undo/redo: last action is first to be undone.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Classic Stack Problem: Valid Parentheses**

```python
def is_valid_parentheses(s: str) -> bool:
    """
    Check if brackets are properly matched and nested.
    O(n) time, O(n) space.

    Think of it like your middleware chain:
    every "opening" must have a matching "closing" in the right order.
    """
    stack: list[str] = []
    matching = {")": "(", "}": "{", "]": "["}

    for char in s:
        if char in "({[":
            stack.append(char)       # Opening bracket: push onto stack
        elif char in ")}]":
            if not stack:
                return False         # Closing bracket with nothing open
            if stack[-1] != matching[char]:
                return False         # Mismatched bracket type
            stack.pop()              # Matched! Remove the opening bracket

    return len(stack) == 0           # All brackets should be matched

# is_valid_parentheses("({[]})")  → True
# is_valid_parentheses("({[}])")  → False (] doesn't match {)
# is_valid_parentheses("((")      → False (unclosed)
```

**Queues: First-In, First-Out (FIFO)**

```
┌─────────────────────────────────────────────────────────────────┐
│                        QUEUE (FIFO)                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Think: a line at a restaurant. First person in line             │
│  is the first person served.                                    │
│                                                                 │
│   enqueue("A") → enqueue("B") → enqueue("C") → dequeue() = "A" │
│                                                                 │
│     front ──▶ ┌───┬───┬───┐ ◀── back                           │
│               │ A │ B │ C │                                     │
│               └───┴───┴───┘                                     │
│                 ▲                                                │
│                 └── first to come out                            │
│                                                                 │
│  In Python: use collections.deque (NOT a list — list.pop(0)     │
│  is O(n), deque.popleft() is O(1))                              │
│                                                                 │
│  from collections import deque                                  │
│  q = deque()                                                    │
│  ├─ enqueue → q.append(item)      O(1)                          │
│  ├─ dequeue → q.popleft()         O(1)                          │
│  └─ peek   → q[0]                 O(1)                          │
│                                                                 │
│  WHERE YOU'VE SEEN THIS:                                        │
│  ├─ Celery task queue (Week 11): tasks processed in order       │
│  ├─ Redis List used as queue: LPUSH + RPOP                      │
│  ├─ Request queue: your API server processes requests in order  │
│  └─ Background job pipeline: jobs enqueued, workers dequeue     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Queue Pattern: BFS (Breadth-First Search)**

```python
from collections import deque

def bfs_shortest_path(
    graph: dict[str, list[str]],
    start: str,
    end: str
) -> list[str] | None:
    """
    Find the shortest path between two nodes using BFS.
    O(V + E) time, O(V) space — where V = vertices, E = edges.

    Backend connection:
    Think of this like finding the shortest chain of service calls
    from Service A to Service B in your architecture diagram
    from Lecture 1 this week.
    """
    queue: deque[list[str]] = deque([[start]])   # Queue of PATHS
    visited: set[str] = {start}

    while queue:
        path = queue.popleft()        # FIFO: explore nearest first
        current = path[-1]

        if current == end:
            return path                # Found shortest path!

        for neighbor in graph.get(current, []):
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(path + [neighbor])

    return None                        # No path exists

# Example: service dependency graph
services = {
    "api_gateway": ["auth_service", "task_service"],
    "auth_service": ["user_db", "redis_cache"],
    "task_service": ["task_db", "notification_service"],
    "notification_service": ["email_service", "redis_cache"],
}

path = bfs_shortest_path(services, "api_gateway", "email_service")
# → ["api_gateway", "task_service", "notification_service", "email_service"]
```

---

## 3.4 Trees — Hierarchical Thinking

```
┌─────────────────────────────────────────────────────────────────┐
│                     TREES IN INTERVIEWS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│              root (CEO)                                         │
│             ╱          ╲                                        │
│          VP Eng       VP Sales                                  │
│         ╱    ╲           ╲                                      │
│      Lead   Lead        Lead                                    │
│      ╱  ╲     ╲                                                 │
│   Dev1  Dev2  Dev3                                              │
│                                                                 │
│  A tree is a graph with NO CYCLES and ONE root.                 │
│  Every node has exactly one parent (except root).               │
│                                                                 │
│  KEY PROPERTIES:                                                │
│  ├─ Depth: distance from root to a node                         │
│  ├─ Height: longest path from a node to a leaf                  │
│  └─ Binary tree: each node has at most 2 children               │
│                                                                 │
│  WHERE YOU'VE SEEN THIS:                                        │
│  ├─ B-tree indexes in PostgreSQL (Week 7)                       │
│  │  └─ Balanced tree → guaranteed O(log n) lookup               │
│  ├─ Your capstone data model: Org → Projects → Tasks            │
│  │  └─ Hierarchical data is a tree!                             │
│  ├─ File system structure (your project directories)            │
│  └─ JSON/dict nesting (your Pydantic nested models)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Tree Traversal — DFS (Depth-First Search)**

```python
from typing import Any

class TreeNode:
    def __init__(self, val: Any, children: list["TreeNode"] | None = None):
        self.val = val
        self.children = children or []

def dfs_collect(root: TreeNode | None) -> list[Any]:
    """
    Collect all values via depth-first traversal.
    O(n) time, O(h) space — h is tree height (recursion depth).

    Backend connection:
    This is how you'd traverse your Org → Project → Task hierarchy
    to generate a full export (your Week 14 CSV export feature).
    """
    if root is None:
        return []

    result = [root.val]
    for child in root.children:
        result.extend(dfs_collect(child))

    return result

# For BINARY trees specifically (the most common in interviews):
class BinaryNode:
    def __init__(
        self,
        val: int,
        left: "BinaryNode | None" = None,
        right: "BinaryNode | None" = None
    ):
        self.val = val
        self.left = left
        self.right = right

def max_depth(root: BinaryNode | None) -> int:
    """
    Find the maximum depth of a binary tree.
    O(n) time — must visit every node.
    O(h) space — recursion stack depth.
    """
    if root is None:
        return 0
    return 1 + max(max_depth(root.left), max_depth(root.right))
```

**Binary Search Tree (BST) — the interview staple:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  BINARY SEARCH TREE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Rule: left child < parent < right child                        │
│                                                                 │
│                    8                                             │
│                  ╱   ╲                                          │
│                3      10                                        │
│              ╱   ╲       ╲                                      │
│             1     6      14                                     │
│                 ╱   ╲   ╱                                       │
│                4     7 13                                       │
│                                                                 │
│  Search: O(log n) average — exactly like binary search          │
│  Insert: O(log n) average — find the right spot, add it        │
│  Delete: O(log n) average — find it, restructure               │
│                                                                 │
│  THIS IS YOUR DATABASE INDEX.                                   │
│  PostgreSQL's B-tree is a balanced version of this.             │
│  When you do WHERE id = 42, the DB walks this tree.             │
│                                                                 │
│  In-order traversal (left → root → right) gives SORTED output: │
│  1, 3, 4, 6, 7, 8, 10, 13, 14                                  │
│  This is how ORDER BY works on an indexed column.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
def is_valid_bst(
    node: BinaryNode | None,
    min_val: float = float("-inf"),
    max_val: float = float("inf")
) -> bool:
    """
    Validate that a binary tree maintains BST property.
    O(n) time — check every node. O(h) space — recursion.

    Interview classic. The trick: each node must be within
    a RANGE defined by its ancestors, not just its parent.
    """
    if node is None:
        return True

    if not (min_val < node.val < max_val):
        return False

    return (
        is_valid_bst(node.left, min_val, node.val)     # Left must be < current
        and is_valid_bst(node.right, node.val, max_val) # Right must be > current
    )
```

---

## 3.5 Graphs — Everything is Connected

```
┌─────────────────────────────────────────────────────────────────┐
│                    GRAPHS IN INTERVIEWS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A graph is nodes (vertices) connected by edges.                │
│  Trees are graphs. BUT graphs can have CYCLES.                  │
│                                                                 │
│      A ────── B                                                 │
│      │ ╲      │                                                │
│      │   ╲    │         Directed graph (edges have direction):  │
│      │     ╲  │         A → B means "A depends on B"            │
│      C ────── D                                                 │
│                         Undirected: A — B means mutual link     │
│                                                                 │
│  REPRESENTATION (the two ways):                                 │
│                                                                 │
│  Adjacency List (most common in interviews):                    │
│  graph = {                                                      │
│      "A": ["B", "C", "D"],                                     │
│      "B": ["A", "D"],                                          │
│      "C": ["A", "D"],                                          │
│      "D": ["A", "B", "C"],                                     │
│  }                                                              │
│                                                                 │
│  WHERE YOU'VE SEEN THIS:                                        │
│  ├─ Alembic migration chain: each migration depends on prior    │
│  ├─ Docker Compose depends_on: service startup order            │
│  ├─ Your system architecture diagram (Lecture 1 this week)      │
│  ├─ FastAPI dependency injection graph (Depends chains)         │
│  └─ User relationships (org members, shared projects)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Topological Sort — you already use this daily:**

```python
from collections import deque

def topological_sort(graph: dict[str, list[str]]) -> list[str] | None:
    """
    Order nodes so that every dependency comes before the node that
    depends on it. Returns None if there's a cycle (impossible to order).

    O(V + E) time, O(V) space.

    Backend connection:
    This is EXACTLY how Alembic decides migration order.
    And how Docker Compose determines service startup order.
    And how your CI/CD pipeline orders build steps.
    """
    # Count incoming edges for each node
    in_degree: dict[str, int] = {node: 0 for node in graph}
    for node in graph:
        for neighbor in graph[node]:
            in_degree[neighbor] = in_degree.get(neighbor, 0) + 1

    # Start with nodes that have no dependencies
    queue: deque[str] = deque(
        node for node, degree in in_degree.items() if degree == 0
    )
    result: list[str] = []

    while queue:
        current = queue.popleft()
        result.append(current)

        for neighbor in graph.get(current, []):
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)  # All dependencies satisfied

    if len(result) != len(in_degree):
        return None  # Cycle detected — can't order!

    return result

# Example: migration dependencies
migrations = {
    "001_create_users": ["002_create_orgs"],
    "002_create_orgs": ["003_create_projects", "004_add_org_roles"],
    "003_create_projects": ["005_create_tasks"],
    "004_add_org_roles": [],
    "005_create_tasks": [],
}
order = topological_sort(migrations)
# → ["001_create_users", "002_create_orgs", "004_add_org_roles",
#    "003_create_projects", "005_create_tasks"]
```

---

## 3.6 The Backend Pattern Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────┐
│        BACKEND INTERVIEW DSA PATTERNS — PRIORITY ORDER          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TIER 1: MASTER THESE (covers ~60% of coding questions)         │
│  ├─ Hash map / set lookup          (Two Sum, frequency, dedup)  │
│  ├─ Two pointers                   (sorted array problems)      │
│  ├─ Sliding window                 (subarray/substring)         │
│  └─ BFS/DFS on graphs and trees    (traversal, shortest path)   │
│                                                                 │
│  TIER 2: KNOW WELL (covers next ~25%)                           │
│  ├─ Stack problems                 (parentheses, monotonic)     │
│  ├─ Binary search                  (sorted data, boundaries)    │
│  ├─ Queue / BFS level-order        (level-by-level processing)  │
│  ├─ Topological sort               (dependency ordering)        │
│  └─ Heap / priority queue          (top-K, merge K lists)       │
│                                                                 │
│  TIER 3: AWARENESS (less common in backend interviews)          │
│  ├─ Dynamic programming            (optimization problems)      │
│  ├─ Backtracking                   (combinations, permutations) │
│  ├─ Trie                           (autocomplete, prefix search)│
│  └─ Union-Find                     (connected components)       │
│                                                                 │
│  FOCUS YOUR TIME ON TIER 1 AND TIER 2.                          │
│  A candidate who nails Tier 1+2 and can discuss tradeoffs       │
│  will outperform one who memorized Tier 3 without understanding │
│  Tier 1.                                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: SQL INTERVIEW MASTERY

## 4.1 Why Backend Interviews Test SQL Hard

> "Frontend engineers get asked about DOM traversal. You get asked about SQL. Every backend interview I've seen for mid-level roles includes at least one SQL question. Your Weeks 5–7 experience gives you a huge advantage — but interviews test one thing you might not have used yet: **window functions**."

```
┌─────────────────────────────────────────────────────────────────┐
│              SQL INTERVIEW CATEGORIES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CATEGORY               YOU ALREADY KNOW     INTERVIEW TWIST   │
│  ─────────              ──────────────────   ───────────────   │
│  JOINs                  ✅ Week 5            Multi-table JOIN   │
│                                               chains, self-joins│
│                                                                 │
│  Aggregation            ✅ Week 5            GROUP BY with      │
│  (GROUP BY, HAVING)                          HAVING filters     │
│                                                                 │
│  Subqueries / CTEs      ✅ Week 5            Correlated         │
│                                               subqueries        │
│                                                                 │
│  Window Functions       ⚠️  NEW TODAY         ROW_NUMBER, RANK, │
│                                               LAG, running sums │
│                                                                 │
│  Data Modification      ✅ Week 5            UPDATE with JOIN,  │
│                                               DELETE duplicates  │
│                                                                 │
│  Schema Design          ✅ Week 5-6          "Design a schema   │
│                                               for X" questions  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Window Functions (The Interview Differentiator)

**This is the one topic that separates candidates in SQL rounds.**

**The core idea:**

> "You know GROUP BY — it collapses rows into groups. Window functions do calculations ACROSS rows WITHOUT collapsing them. Every row stays. You get the original data PLUS the computed value."

```
┌─────────────────────────────────────────────────────────────────┐
│              GROUP BY vs WINDOW FUNCTION                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ORIGINAL DATA (employee_salaries):                             │
│  ┌────────────┬────────────┬─────────┐                          │
│  │ name       │ department │ salary  │                          │
│  ├────────────┼────────────┼─────────┤                          │
│  │ Alice      │ Eng        │ 90,000  │                          │
│  │ Bob        │ Eng        │ 85,000  │                          │
│  │ Charlie    │ Eng        │ 95,000  │                          │
│  │ Diana      │ Sales      │ 70,000  │                          │
│  │ Eve        │ Sales      │ 80,000  │                          │
│  └────────────┴────────────┴─────────┘                          │
│                                                                 │
│  GROUP BY — collapses rows:                                     │
│  SELECT department, AVG(salary) FROM emp GROUP BY department;   │
│  ┌────────────┬─────────┐                                       │
│  │ department │ avg_sal │  ← Only 2 rows. Names are GONE.      │
│  ├────────────┼─────────┤                                       │
│  │ Eng        │ 90,000  │                                       │
│  │ Sales      │ 75,000  │                                       │
│  └────────────┴─────────┘                                       │
│                                                                 │
│  WINDOW FUNCTION — keeps all rows:                              │
│  SELECT name, department, salary,                               │
│         AVG(salary) OVER(PARTITION BY department) AS dept_avg   │
│  FROM emp;                                                      │
│  ┌────────────┬────────────┬─────────┬──────────┐               │
│  │ name       │ department │ salary  │ dept_avg │               │
│  ├────────────┼────────────┼─────────┼──────────┤               │
│  │ Alice      │ Eng        │ 90,000  │ 90,000   │               │
│  │ Bob        │ Eng        │ 85,000  │ 90,000   │               │
│  │ Charlie    │ Eng        │ 95,000  │ 90,000   │               │
│  │ Diana      │ Sales      │ 70,000  │ 75,000   │               │
│  │ Eve        │ Sales      │ 80,000  │ 75,000   │               │
│  └────────────┴────────────┴─────────┴──────────┘               │
│                All 5 rows preserved! Each gets the group average.│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The window function anatomy:**

```sql
-- THE SYNTAX:
function_name() OVER (
    PARTITION BY column    -- Like GROUP BY, but doesn't collapse
    ORDER BY column        -- Order within each partition
    ROWS BETWEEN ...       -- Optional: define the window frame
)
```

```
┌─────────────────────────────────────────────────────────────────┐
│              WINDOW FUNCTION ANATOMY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   AVG(salary)  OVER  (PARTITION BY department ORDER BY salary)  │
│       │          │         │                       │            │
│       │          │         │                       │            │
│   What to       │    Split into groups         Order within    │
│   calculate     │    (like GROUP BY,           each group      │
│                 │     but keep rows)                            │
│                 │                                               │
│            "Over this                                           │
│             window of                                           │
│             rows"                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The essential window functions for interviews:**

### ROW_NUMBER(), RANK(), DENSE_RANK()

```sql
-- All three assign a RANKING to each row within a partition.
-- The difference is how they handle TIES.

SELECT
    name,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
    RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank
FROM employee_salaries;
```

```
┌──────────┬──────┬────────┬─────────┬──────┬────────────┐
│ name     │ dept │ salary │ row_num │ rank │ dense_rank │
├──────────┼──────┼────────┼─────────┼──────┼────────────┤
│ Charlie  │ Eng  │ 95,000 │    1    │  1   │     1      │
│ Alice    │ Eng  │ 90,000 │    2    │  2   │     2      │
│ Bob      │ Eng  │ 85,000 │    3    │  3   │     3      │
│ Eve      │ Sales│ 80,000 │    1    │  1   │     1      │
│ Diana    │ Sales│ 70,000 │    2    │  2   │     2      │
└──────────┴──────┴────────┴─────────┴──────┴────────────┘

Now, what if Alice and Charlie both earned 95,000?

┌──────────┬──────┬────────┬─────────┬──────┬────────────┐
│ name     │ dept │ salary │ row_num │ rank │ dense_rank │
├──────────┼──────┼────────┼─────────┼──────┼────────────┤
│ Charlie  │ Eng  │ 95,000 │    1    │  1   │     1      │  ← tie
│ Alice    │ Eng  │ 95,000 │    2    │  1   │     1      │  ← tie
│ Bob      │ Eng  │ 85,000 │    3    │  3   │     2      │
│                            ▲         ▲        ▲
│                            │         │        │
│                  Always    │  Skips   │  Never
│                  unique    │  rank 2  │  skips
│                            │ (1,1,3)  │ (1,1,2)
└──────────────────────────────────────────────────────────┘

USE ROW_NUMBER when: you need unique numbering (pagination, dedup)
USE RANK when:       ties should share rank, gaps are OK
USE DENSE_RANK when: ties should share rank, NO gaps
```

### LAG() and LEAD()

```sql
-- Access the PREVIOUS row (LAG) or NEXT row (LEAD) without a self-join.
-- Interview gold: "Compare each row to the one before/after it."

SELECT
    date,
    revenue,
    LAG(revenue, 1)  OVER (ORDER BY date) AS prev_day_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY date) AS daily_change,
    LEAD(revenue, 1) OVER (ORDER BY date) AS next_day_revenue
FROM daily_revenue;
```

```
┌────────────┬─────────┬──────────────────┬──────────────┐
│ date       │ revenue │ prev_day_revenue │ daily_change │
├────────────┼─────────┼──────────────────┼──────────────┤
│ 2025-01-01 │ 1,000   │ NULL             │ NULL         │
│ 2025-01-02 │ 1,200   │ 1,000            │ +200         │
│ 2025-01-03 │ 900     │ 1,200            │ -300         │
│ 2025-01-04 │ 1,500   │ 900              │ +600         │
└────────────┴─────────┴──────────────────┴──────────────┘

LAG(column, n) = look back n rows (default 1)
LEAD(column, n) = look forward n rows (default 1)
First row's LAG is NULL (no previous). Last row's LEAD is NULL.
```

### Running Totals with SUM() OVER

```sql
-- A cumulative sum that grows row by row.
-- Interview classic: "Show running total of sales."

SELECT
    date,
    amount,
    SUM(amount) OVER (ORDER BY date) AS running_total,
    SUM(amount) OVER (
        PARTITION BY EXTRACT(MONTH FROM date)
        ORDER BY date
    ) AS monthly_running_total
FROM transactions;
```

```
┌────────────┬────────┬───────────────┬────────────────────────┐
│ date       │ amount │ running_total │ monthly_running_total  │
├────────────┼────────┼───────────────┼────────────────────────┤
│ 2025-01-01 │   100  │     100       │     100                │
│ 2025-01-15 │   200  │     300       │     300                │
│ 2025-01-28 │   150  │     450       │     450                │
│ 2025-02-01 │   300  │     750       │     300  ← reset       │
│ 2025-02-10 │   100  │     850       │     400                │
└────────────┴────────┴───────────────┴────────────────────────┘

ORDER BY inside OVER() creates a "growing window" —
each row includes all rows up to and including itself.

PARTITION BY resets the window for each group.
```

---

## 4.3 Common SQL Interview Patterns

### Pattern 1: "Find the Nth highest salary"

```sql
-- "Find the 3rd highest salary in each department."
-- This is the #1 most-asked SQL interview question.

WITH ranked AS (
    SELECT
        name,
        department,
        salary,
        DENSE_RANK() OVER (
            PARTITION BY department
            ORDER BY salary DESC
        ) AS rank
    FROM employees
)
SELECT name, department, salary
FROM ranked
WHERE rank = 3;

-- WHY DENSE_RANK and not ROW_NUMBER?
-- If two people tie for 2nd, the next person should be 3rd,
-- not 4th. DENSE_RANK handles this correctly.
```

### Pattern 2: "Find duplicates and keep only one"

```sql
-- "Delete duplicate rows, keeping the one with the lowest ID."

WITH ranked AS (
    SELECT
        id,
        email,
        ROW_NUMBER() OVER (
            PARTITION BY email       -- Group by the duplicate key
            ORDER BY id ASC          -- Keep lowest ID (row_num = 1)
        ) AS rn
    FROM users
)
DELETE FROM users
WHERE id IN (SELECT id FROM ranked WHERE rn > 1);

-- ROW_NUMBER guarantees unique numbers even for ties.
-- row_num = 1 is the "keeper." Everything else is deleted.
```

### Pattern 3: "Year-over-year comparison"

```sql
-- "Show each month's revenue alongside the same month last year."

WITH monthly AS (
    SELECT
        DATE_TRUNC('month', created_at) AS month,
        SUM(amount) AS revenue
    FROM orders
    GROUP BY DATE_TRUNC('month', created_at)
)
SELECT
    month,
    revenue,
    LAG(revenue, 12) OVER (ORDER BY month) AS same_month_last_year,
    ROUND(
        (revenue - LAG(revenue, 12) OVER (ORDER BY month))
        / LAG(revenue, 12) OVER (ORDER BY month) * 100,
        1
    ) AS yoy_change_pct
FROM monthly;
```

### Pattern 4: "Top N per group"

```sql
-- "Find the top 2 highest-paid employees in each department."
-- This CANNOT be done with GROUP BY. Window functions required.

WITH ranked AS (
    SELECT
        name,
        department,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY department
            ORDER BY salary DESC
        ) AS rn
    FROM employees
)
SELECT name, department, salary
FROM ranked
WHERE rn <= 2;
```

---

## 4.4 The SQL Problem-Solving Framework

```
┌─────────────────────────────────────────────────────────────────┐
│           SQL INTERVIEW PROBLEM-SOLVING STEPS                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: UNDERSTAND THE OUTPUT                                  │
│  ├─ "What columns should the result have?"                      │
│  ├─ "One row per what? Per user? Per month? Per department?"    │
│  └─ This tells you what to GROUP BY or PARTITION BY             │
│                                                                 │
│  STEP 2: IDENTIFY THE JOINS                                     │
│  ├─ "How many tables do I need?"                                │
│  ├─ "What's the relationship?" (your ER diagrams from Week 5)  │
│  └─ Start with FROM + JOINs before any calculations            │
│                                                                 │
│  STEP 3: FILTER FIRST (WHERE)                                   │
│  ├─ Reduce the dataset before expensive operations              │
│  └─ WHERE happens BEFORE GROUP BY and window functions          │
│                                                                 │
│  STEP 4: CHOOSE YOUR WEAPON                                     │
│  ├─ Need to collapse rows?    → GROUP BY + aggregate            │
│  ├─ Need per-row calculation  → Window function                 │
│  │   with group context?                                        │
│  ├─ Need ranking?             → ROW_NUMBER / RANK / DENSE_RANK │
│  ├─ Need previous/next row?   → LAG / LEAD                     │
│  └─ Need running total?       → SUM() OVER(ORDER BY ...)       │
│                                                                 │
│  STEP 5: USE CTEs FOR CLARITY                                   │
│  ├─ One CTE per logical step                                    │
│  ├─ Name them clearly (ranked, filtered, aggregated)            │
│  └─ Interviewers LOVE readable SQL. CTEs help you and them.    │
│                                                                 │
│  SQL ORDER OF EXECUTION (know this cold):                       │
│  FROM → JOIN → WHERE → GROUP BY → HAVING                        │
│  → Window Functions → SELECT → DISTINCT → ORDER BY → LIMIT     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: BEHAVIORAL & PORTFOLIO

## 5.1 The STAR Method (Commentary Driving)

> "Remember the driving test analogy? Behavioral interviews are like recounting your driving history. The interviewer asks: 'Tell me about a time you handled a difficult situation.' They want a STRUCTURED story, not a rambling anecdote."

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE STAR METHOD                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  S — SITUATION  (Set the scene. 2-3 sentences.)                 │
│  "During our capstone project, we were building a multi-tenant  │
│   SaaS backend with real-time notifications..."                 │
│                                                                 │
│  T — TASK  (What was YOUR responsibility?)                      │
│  "I was responsible for designing the WebSocket notification    │
│   system that needed to scale across multiple server instances" │
│                                                                 │
│  A — ACTION  (What did YOU specifically do? Most of your time.) │
│  "I researched the scaling problem, wrote an ADR comparing      │
│   three approaches, implemented Redis Pub/Sub to broadcast      │
│   across instances, and built a connection manager with         │
│   heartbeat detection..."                                       │
│                                                                 │
│  R — RESULT  (Quantify if possible.)                            │
│  "The system handled 500 concurrent connections with sub-200ms  │
│   delivery latency. The ADR was referenced by the team for      │
│   future architectural decisions."                              │
│                                                                 │
│  TIME ALLOCATION:                                               │
│  ├─ S: 10% (brief context)                                     │
│  ├─ T: 10% (your role)                                         │
│  ├─ A: 60% (the meat — what YOU did and WHY)                   │
│  └─ R: 20% (outcome + what you learned)                        │
│                                                                 │
│  TOTAL: 2-3 minutes per answer. Not 30 seconds. Not 10 minutes.│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Mining Your Capstone for Stories

**You already have every story you need. Your ADRs from Week 14 are pre-written STAR answers.**

```
┌─────────────────────────────────────────────────────────────────┐
│          YOUR CAPSTONE → INTERVIEW STORIES                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INTERVIEW QUESTION              YOUR CAPSTONE STORY            │
│  ──────────────────              ──────────────────             │
│  "Tell me about a                Your schema design for         │
│   technical decision              multi-tenancy — row-level     │
│   you had to make"                vs schema-per-tenant.         │
│                                   You wrote an ADR for this.    │
│                                                                 │
│  "How do you handle              N+1 query problem you found    │
│   performance issues?"            and fixed. You have EXPLAIN   │
│                                   before/after numbers.         │
│                                                                 │
│  "Describe a time you            JWT refresh token rotation     │
│   improved security"              and Redis-backed revocation.  │
│                                                                 │
│  "Tell me about a                Database migration that        │
│   mistake you made"               needed a rollback strategy.   │
│                                   Your post-mortem covers this. │
│                                                                 │
│  "How do you handle              Celery task retry with         │
│   failures in a system?"          exponential backoff and       │
│                                   idempotency guarantees.       │
│                                                                 │
│  "Describe working               PR reviews, git workflow,      │
│   with a codebase"                reading existing code.        │
│                                                                 │
│  YOUR ADRs AND POST-MORTEM ARE PRE-WRITTEN STAR ANSWERS.       │
│  Review them before every interview.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 The "Walk Me Through Your Project" Framework

**You WILL be asked this. Have a structured 5-minute walkthrough ready.**

```
┌─────────────────────────────────────────────────────────────────┐
│         THE PROJECT WALKTHROUGH STRUCTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LAYER 1: THE WHAT (30 seconds)                                 │
│  "I built a multi-tenant project management SaaS backend.       │
│   Users can create organizations, manage projects and tasks,    │
│   with real-time notifications and role-based access control."  │
│                                                                 │
│  LAYER 2: THE ARCHITECTURE (60 seconds)                         │
│  "It's a FastAPI application with PostgreSQL for persistence,   │
│   Redis for caching and pub/sub, and Celery for background      │
│   jobs. I used async SQLAlchemy throughout for non-blocking     │
│   database access."                                             │
│   [Draw the architecture diagram if whiteboard available]       │
│                                                                 │
│  LAYER 3: THE INTERESTING DECISIONS (2-3 minutes)               │
│  Pick 2-3 decisions you made AND WHY.                           │
│  "The most interesting challenge was multi-tenancy. I chose     │
│   row-level isolation over schema-per-tenant because [reasons]. │
│   I enforced isolation through a middleware dependency that      │
│   automatically scopes all queries to the current tenant."      │
│                                                                 │
│  LAYER 4: THE RESULTS (30 seconds)                              │
│  "The API handles 500 requests/minute with sub-200ms p95        │
│   latency. I have 80%+ test coverage with both unit and        │
│   integration tests, and it's deployed with CI/CD on [cloud]."  │
│                                                                 │
│  LAYER 5: THE REFLECTION (30 seconds)                           │
│  "If I were to do it again, I'd [specific improvement].         │
│   I documented this in my post-mortem."                         │
│                                                                 │
│  KEY RULES:                                                     │
│  ├─ NEVER just list technologies. Explain DECISIONS.            │
│  ├─ Always include a tradeoff: "I chose X over Y because..."   │
│  ├─ Include numbers: latency, coverage, concurrent users.       │
│  └─ End with self-awareness: what you'd improve shows growth.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 Common Behavioral Mistakes

```
┌─────────────────────────────────────────────────────────────────┐
│              BEHAVIORAL INTERVIEW MISTAKES                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MISTAKE 1: "We did X, we built Y, we decided Z"               │
│  ─────────                                                      │
│  Interviewers want to hear what YOU did, not your team.         │
│  Say "I" not "we." It's not arrogance — it's clarity.           │
│  "I designed the schema. I identified the N+1 problem.          │
│   I proposed the caching strategy in an ADR."                   │
│                                                                 │
│  MISTAKE 2: Technology shopping list                            │
│  ─────────                                                      │
│  "I used FastAPI, PostgreSQL, Redis, Celery, Docker..."         │
│  This says NOTHING. Anyone can list tools.                      │
│  Instead: "I chose Redis for session storage because we needed  │
│  fast TTL-based expiration for refresh tokens, and Redis's      │
│  atomic INCR gave us rate limiting for free."                   │
│                                                                 │
│  MISTAKE 3: No conflict or challenge                            │
│  ─────────                                                      │
│  "Everything went smoothly." ← This is a red flag.             │
│  Interviewers want to hear how you handle DIFFICULTY.           │
│  Your post-mortem from the capstone is full of honest           │
│  challenges. Use them.                                          │
│                                                                 │
│  MISTAKE 4: No quantified results                               │
│  ─────────                                                      │
│  "It was faster." → "Latency dropped from 800ms to 120ms."     │
│  "I improved it." → "Query time improved 10x after indexing."   │
│  You have these numbers from your load tests and EXPLAIN output.│
│  Use them.                                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 6: THE PRACTICE STRATEGY

## 6.1 The 80/20 Rule for Backend DSA

```
┌─────────────────────────────────────────────────────────────────┐
│              EFFICIENT PRACTICE STRATEGY                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DON'T DO THIS:                                                 │
│  ├─ ❌ Random LeetCode problems ("I'll just do 500")            │
│  ├─ ❌ Only hard problems (ego-driven, low return)              │
│  ├─ ❌ Memorizing solutions (fails on any variation)            │
│  └─ ❌ Only DSA, ignoring SQL and system design                 │
│                                                                 │
│  DO THIS:                                                       │
│  ├─ ✅ Study by PATTERN, not by problem                         │
│  │    └─ Do 3-5 hash map problems in a row.                    │
│  │       Then 3-5 BFS/DFS. Then sliding window. Etc.           │
│  │       You're training pattern recognition, not memorization. │
│  │                                                              │
│  ├─ ✅ Mix difficulties: 40% Easy, 50% Medium, 10% Hard         │
│  │    └─ Easies build confidence and speed.                    │
│  │       Mediums are what you'll face in interviews.            │
│  │       Hards are rare and often unfair.                       │
│  │                                                              │
│  ├─ ✅ Practice TALKING OUT LOUD                                 │
│  │    └─ Solve problems while narrating. Record yourself.       │
│  │       This feels absurd. Do it anyway. The interview         │
│  │       is a communication exercise, not a coding one.        │
│  │                                                              │
│  ├─ ✅ Allocate time across all round types                      │
│  │    ├─ DSA: 40% of prep time                                  │
│  │    ├─ SQL: 20% of prep time                                  │
│  │    ├─ System design: 20% of prep time (Lectures 1-2 today)  │
│  │    └─ Behavioral: 20% of prep time (review your ADRs/PM)    │
│  │                                                              │
│  └─ ✅ Time-box each problem: 30 min max                        │
│       └─ Can't solve in 30 min? Study the solution.            │
│          Understand the PATTERN. Redo it in 2 days              │
│          without looking (spaced retrieval — Week 2 concepts).  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A concrete weekly study rhythm for job searching:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WEEKLY STUDY TEMPLATE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MONDAY:    2 LeetCode (same pattern)    + 1 SQL problem        │
│  TUESDAY:   2 LeetCode (same pattern)    + Review ADR stories   │
│  WEDNESDAY: 2 LeetCode (NEW pattern)     + 1 SQL problem        │
│  THURSDAY:  1 System Design practice     + 1 Mock behavioral    │
│  FRIDAY:    REDO 2 problems from Mon/Tue (spaced retrieval)     │
│  WEEKEND:   1 Mock interview (full loop) OR rest                │
│                                                                 │
│  Total: ~8 DSA, 2 SQL, 1 System Design, 1 Behavioral per week  │
│  Time: ~1.5 hours/day during active job search                  │
│                                                                 │
│  RESOURCES:                                                     │
│  ├─ DSA:   NeetCode 150 (curated, pattern-organized)            │
│  ├─ SQL:   LeetCode SQL 50 or HackerRank SQL track              │
│  ├─ SysD:  Practice by redesigning systems you know             │
│  │         (How would you build Twitter? Uber? Slack?)          │
│  └─ Mock:  Pramp (free), interviewing.io, or a friend           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 The Interview Day Playbook

```
┌─────────────────────────────────────────────────────────────────┐
│              INTERVIEW DAY PROTOCOL                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE:                                                        │
│  ├─ Review your STAR stories (ADRs + post-mortem)               │
│  ├─ Review your project walkthrough (practice out loud)          │
│  ├─ Warm up with 1 easy LeetCode problem (15 min, not more)    │
│  └─ Have water. Eat. Sleep well the night before.               │
│                                                                 │
│  DURING A CODING ROUND:                                         │
│  ├─ Use the 5-Step Protocol (Section 1.4)                       │
│  ├─ If stuck: "Let me think about what data structure           │
│  │   would help here..." (buy time, show process)               │
│  ├─ If STILL stuck: "I'm considering [approach] because...      │
│  │   Can I get a hint on whether I'm on the right track?"       │
│  ├─ Write clean Python. Use type hints. You do this naturally.  │
│  └─ ALWAYS state complexity at the end, unprompted.              │
│                                                                 │
│  DURING A SQL ROUND:                                            │
│  ├─ Clarify the schema first. Draw it if you can.              │
│  ├─ Use CTEs for readability.                                   │
│  ├─ State whether you'd add indexes for performance.            │
│  └─ Window functions are your differentiator. Use them.         │
│                                                                 │
│  DURING A SYSTEM DESIGN ROUND:                                  │
│  ├─ You covered this in Lectures 1-2 this week.                │
│  ├─ Start with requirements, then high-level design,            │
│  │   then deep dive into one component.                         │
│  └─ Your capstone architecture IS a system design answer.       │
│                                                                 │
│  DURING BEHAVIORAL:                                             │
│  ├─ STAR method. Every answer. No exceptions.                   │
│  ├─ "I" not "we."                                               │
│  ├─ Include a tradeoff or challenge.                            │
│  └─ End with what you learned.                                  │
│                                                                 │
│  AFTER:                                                         │
│  ├─ Send a thank-you email (brief, genuine).                    │
│  ├─ Write down every question you remember.                     │
│  ├─ Review problems you struggled with.                         │
│  └─ Don't catastrophize. Even great candidates bomb rounds.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.3 Continued Growth After This Course

```
┌─────────────────────────────────────────────────────────────────┐
│             THE ROAD AHEAD                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You now have:                                                  │
│  ├─ A production-grade project with real architecture decisions │
│  ├─ Experience with async, databases, caching, auth, queues    │
│  ├─ CI/CD, Docker, cloud deployment — production skills        │
│  ├─ ADRs and a post-mortem — ready-made interview stories      │
│  ├─ Load test results — quantified performance numbers          │
│  └─ This framework for focused interview preparation           │
│                                                                 │
│  WHAT COMES NEXT:                                               │
│  ├─ DSA practice: 75+ problems → aim for 150+ during search    │
│  ├─ System design: revisit Lectures 1-2, practice weekly       │
│  ├─ SQL: window functions alone set you apart. Master them.    │
│  ├─ Mock interviews: practice with humans, not just screens    │
│  └─ Ship side projects: each one is a new portfolio story      │
│                                                                 │
│  THE REAL SECRET:                                               │
│  You have something most candidates don't — you've BUILT       │
│  the system, not just studied it. When an interviewer asks      │
│  "how would you design a caching layer?", you don't theorize.  │
│  You say: "I built one. Here's what I learned."                │
│                                                                 │
│  That's your advantage. Use it.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              DSA & INTERVIEW QUICK REFERENCE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  COMPLEXITY CHEAT SHEET:                                        │
│      O(1)       dict lookup, array index, Redis GET             │
│      O(log n)   binary search, B-tree index, sorted set lookup  │
│      O(n)       single loop, linear scan, Seq Scan              │
│      O(n log n) sorting (sorted(), ORDER BY)                    │
│      O(n²)      nested loop, N+1 queries, brute force pairs    │
│                                                                 │
│  PATTERN RECOGNITION — FIRST QUESTIONS:                         │
│      "Need to search/match?"          → Hash map                │
│      "Need ordering (FIFO/LIFO)?"     → Queue / Stack           │
│      "Data is hierarchical?"          → Tree traversal          │
│      "Items have relationships?"      → Graph BFS/DFS           │
│      "Looking for contiguous range?"  → Sliding window          │
│      "Data is sorted?"                → Binary search            │
│                                                                 │
│  SQL WINDOW FUNCTIONS:                                          │
│      ROW_NUMBER() OVER (PARTITION BY x ORDER BY y)              │
│      RANK() / DENSE_RANK() for ties                             │
│      LAG(col, n) / LEAD(col, n) for prev/next                  │
│      SUM(col) OVER (ORDER BY x) for running totals             │
│                                                                 │
│  INTERVIEW PROTOCOL:                                            │
│      1. Clarify  →  2. Example  →  3. Approach                  │
│      4. Code     →  5. Verify + state complexity                │
│                                                                 │
│  BEHAVIORAL (STAR):                                             │
│      Situation (10%) → Task (10%) → Action (60%) → Result (20%) │
│      Use "I" not "we." Include tradeoffs. Quantify results.    │
│                                                                 │
│  PREP TIME ALLOCATION:                                          │
│      DSA 40% | SQL 20% | System Design 20% | Behavioral 20%    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  INTERVIEW = DRIVING TEST                                       │
│                                                                 │
│  You're a competent engineer. The interview is a specific       │
│  FORMAT that tests specific SKILLS under specific CONDITIONS.   │
│  Study the format. Practice the skills. Trust the experience.   │
│                                                                 │
│  ┌───────────────────┐  ┌───────────────────┐                   │
│  │ Your 16 Weeks of  │  │ The Interview     │                   │
│  │ Building          │  │ Format            │                   │
│  │                   │  │                   │                   │
│  │ • SaaS backend    │  │ • DSA patterns    │                   │
│  │ • Real decisions  │  │ • SQL queries     │                   │
│  │ • Production code │  │ • System design   │                   │
│  │ • Load tests      │  │ • STAR stories    │                   │
│  │ • ADRs & PM       │  │ • Communication   │                   │
│  └────────┬──────────┘  └──────┬────────────┘                   │
│           │                    │                                │
│           └────────┬───────────┘                                │
│                    │                                            │
│            ┌───────▼───────┐                                    │
│            │   THE BRIDGE  │                                    │
│            │               │                                    │
│            │  This lecture │                                    │
│            │  connects     │                                    │
│            │  what you've  │                                    │
│            │  BUILT to     │                                    │
│            │  what they    │                                    │
│            │  ASK.         │                                    │
│            │               │                                    │
│            └───────────────┘                                    │
│                                                                 │
│  FIVE ENDURING IDEAS:                                           │
│  1. Every line of code has a cost. You can REASON about it.     │
│  2. Data structure choice is the biggest performance lever.     │
│  3. Interview problems are ABSTRACTED versions of real          │
│     engineering — which you've already done.                    │
│  4. Your project experience is your strongest asset —           │
│     if you can ARTICULATE it clearly.                           │
│  5. Consistent, PATTERN-BASED practice beats random grinding.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Final Deliverable & Beyond

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THIS WEEK:                                                     │
│  └─ Final Deliverable presentation                              │
│     Practice your Layer 1-5 project walkthrough (Section 5.3).  │
│     This IS a mock portfolio presentation.                      │
│     Treat it like an interview: time yourself, narrate          │
│     your decisions, state your tradeoffs, show your numbers.    │
│                                                                 │
│  YOUR JOB SEARCH:                                               │
│  ├─ DSA: Continue the weekly rhythm (Section 6.1).              │
│  │  Aim for 150+ problems, organized by pattern.                │
│  ├─ SQL: Window functions WILL appear. Practice them weekly.    │
│  ├─ System Design: Lectures 1-2 this week gave you the          │
│  │  framework. Practice one design per week.                    │
│  ├─ Behavioral: Your ADRs and post-mortem are gold.             │
│  │  Review them before every interview.                         │
│  └─ Portfolio: Your GitHub has a deployed, tested, CI/CD-piped  │
│     SaaS backend. That puts you ahead of most junior/mid        │
│     candidates. Make sure your README tells the story.          │
│                                                                 │
│  YOUR CAREER:                                                   │
│  └─ The skills from this course compound.                       │
│     Complexity analysis helps you write better production code. │
│     System design thinking helps you architect real systems.    │
│     The STAR method helps you in promotion discussions.         │
│     These aren't just interview tricks. They're engineering     │
│     fundamentals that pay dividends for your entire career.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```