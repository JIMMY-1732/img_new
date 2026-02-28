# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "NoSQL is the modern, flexible alternative to SQL. 'Schema-less' means I don't need to think about data structure. MongoDB is basically PostgreSQL but without the rigid table structure. Redis is just a fast cache you bolt on top. When data looks like JSON, use MongoDB — it's simpler."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---|---|
| 1 | "Schema-less means no schema discipline needed" | The schema moves from the database to your application code — without Pydantic, you get silent field-name drift that the DB will never warn you about |
| 2 | "Embedding documents eliminates data integrity concerns" | Embedding **copies** data at insertion time; every collection that contains an embedded copy must be updated independently when that data changes |
| 3 | "Referencing in MongoDB gives you the same safety as PostgreSQL foreign keys" | MongoDB references are ordinary string values — nothing prevents referencing a document that has been deleted; the database raises no error |
| 4 | "`ZADD` accumulates scores like a counter (`INCR`)" | `ZADD` **sets** a score absolutely; calling `ZADD leaderboard 6 charlie` when charlie already has score 7 results in score 6, not 13 |
| 5 | "Redis data persists by default like PostgreSQL" | Default Redis is pure in-memory with zero persistence; without RDB or AOF configuration, every key vanishes on restart |
| 6 | "Any JSON-shaped data belongs in MongoDB, not PostgreSQL" | PostgreSQL JSONB handles semi-structured data while keeping ACID, JOINs, and FK constraints; MongoDB only wins when the **entire model** is document-shaped and cross-collection JOINs are rare |
| 7 | "NoSQL is categorically faster than PostgreSQL" | Performance depends on access patterns and indexes; a PostgreSQL query with the right B-tree index routinely outperforms MongoDB for structured relational queries |

## The Main Purpose (Essence)

**Core Insight:** Every NoSQL database solves one specific pain point of relational databases by deliberately trading away something else. The right question is never "SQL or NoSQL" — it is **"which guarantees does my specific problem require, and which can I afford to give up?"**

| Problem | NoSQL "Solves" It By... | What You Surrender |
|---|---|---|
| Varying product attributes per type | Document store: schema-free documents | DB-enforced validation; Pydantic must do this instead |
| Sub-millisecond counters and leaderboards | Redis: in-memory data structures | Durability; data is lost on restart unless RDB/AOF is configured |
| Avoiding JOINs for co-located reads | Embedding: copy related data inside the parent | Single-place updates; embedded copies must be updated across every document that contains them |
| Horizontal scale across many servers | Denormalization and sharding | Data consistency; duplicate data can silently diverge |

**The Key Distinction:**

| | JSONB in PostgreSQL | MongoDB | Redis |
|---|---|---|---|
| **Best for** | Mostly relational data with some flexible parts | Entire model is document-shaped, JOINs rare | Sub-millisecond HOT data: counters, sessions, leaderboards |
| **Schema** | Enforced by DB for fixed cols; flexible inside JSONB | Enforced by YOUR code (Pydantic) — or not at all | Keys are arbitrary; no schema |
| **Source of truth?** | Yes | Yes | No — always a performance layer on top |

## Prediction Collision Map

| Naive Belief | Confident Wrong Prediction | Actual Behaviour | Exercise That Exposes It |
|---|---|---|---|
| "All docs in a collection share the same field names" | Querying `warranty_years` finds all 3 products | Only 1 found — two have `warrantyYears` / `warranty` | **1.1** |
| "Embedding means one update propagates everywhere" | Renaming Alice in the tasks collection updates her comments | Comments still show "Alice Smith"; they are independent copies | **1.2** |
| "`ZADD key 6 charlie` after `ZADD key 7 charlie` → score 13" | Charlie's score increases to 13 | Charlie's score becomes **6** (ZADD replaces, not increments) | **1.3** |
| "The `->>` operator in JSONB does numeric comparison" | A 6.7" phone is filtered out when `min_inches=13.0` | Phone 6.7 **passes** the filter and sorts **first** because `"6.7" > "13.0"` is True lexicographically | **2.1** |
| "Deleting a user cleans up their associated tasks (like ON DELETE SET NULL)" | Deleting Alice removes her from task assignments | Tasks still hold `assignee_id: "u_001"`; dashboard crashes on `None["name"]` | **2.2** |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Three Developers (Schema Drift)

**Context:** A small team stores electronics product data in a document collection (simulated as a Python list of dicts). In a real MongoDB setup this list is a collection and each dict is a document. The database accepts any field name — no schema is enforced.

```python
# Simulating a MongoDB-style collection.
# In real MongoDB: db.products.insertOne({...}) accepts any dict.
# There is no schema enforcement at the database layer.

products_collection: list[dict] = []

def insert_product(document: dict) -> None:
    """Insert any dict. No field-name validation. No type checking."""
    products_collection.append(document)

def find_with_long_warranty(min_years: int) -> list[dict]:
    """Find products with warranty >= min_years."""
    return [
        p for p in products_collection
        if p.get("warranty_years", 0) >= min_years
    ]

# --- Week 1: First developer ---
insert_product({
    "name": "ThinkPad X1",
    "category": "electronics",
    "price": 1299.99,
    "warranty_years": 3,        # snake_case
})

# --- Week 2: Second developer (different naming convention) ---
insert_product({
    "name": "MacBook Pro",
    "category": "electronics",
    "price": 2499.99,
    "warrantyYears": 2,         # camelCase — different key!
})

# --- Week 3: Third developer (abbreviated) ---
insert_product({
    "name": "Dell XPS 15",
    "category": "electronics",
    "price": 1799.99,
    "warranty": 1,              # abbreviated — yet another key!
})

results = find_with_long_warranty(min_years=1)

print(f"Products with warranty >= 1 year: {len(results)}")
for r in results:
    print(f"  - {r['name']}")
```

**Questions:**

1. **Predict the exact output.** How many products does the function return? Which ones?

2. **Mental Trace.** Walk through `find_with_long_warranty` for each of the three documents individually. For each one: what does `p.get("warranty_years", 0)` return, and does it pass the `>= 1` condition?

3. **The PostgreSQL Contrast.** If these were rows in a PostgreSQL table with a `warranty_years INTEGER NOT NULL` column, could this specific problem even occur? Identify the exact mechanism PostgreSQL uses to prevent it, and explain what a developer **gives up** when moving to a schema-free document store.

4. **The Partial Fix.** A teammate proposes enforcing the schema at the application layer with Pydantic:
   ```python
   from pydantic import BaseModel, Field

   class ElectronicsProduct(BaseModel):
       name: str
       price: float = Field(gt=0)
       warranty_years: int = Field(ge=0)

   def insert_product(document: dict) -> None:
       ElectronicsProduct(**document)  # Validates before inserting
       products_collection.append(document)
   ```
   Does this fully solve the schema drift problem? Describe **two scenarios** where the problem can still occur even after this fix is applied.

---

## Exercise 1.2: The Embedded Name That Diverged

**Context:** A developer built a task system using the "embed everything" philosophy from a NoSQL tutorial. Author display names are embedded inside comments for fast reads — no lookup needed.

```python
comments_collection: list[dict] = []
tasks_collection: list[dict] = []

# --- Initial data setup ---
tasks_collection.append({
    "_id": "task_001",
    "title": "Design the API schema",
    "assignee": {
        "user_id": "user_alice",
        "display_name": "Alice Smith",     # Embedded copy
    }
})

comments_collection.extend([
    {
        "_id": "comment_001",
        "task_id": "task_001",
        "author": {
            "user_id": "user_alice",
            "display_name": "Alice Smith",  # Another independent copy
        },
        "body": "I'll start with the endpoints.",
    },
    {
        "_id": "comment_002",
        "task_id": "task_001",
        "author": {
            "user_id": "user_alice",
            "display_name": "Alice Smith",  # Yet another independent copy
        },
        "body": "Should we use UUID or integer PKs?",
    },
    {
        "_id": "comment_003",
        "task_id": "task_001",
        "author": {
            "user_id": "user_bob",
            "display_name": "Bob Chen",
        },
        "body": "UUIDs — easier for distributed systems.",
    },
])

# Alice gets married. Her display name is updated in the users table (PostgreSQL).
# The developer also updates the task assignee field.
for task in tasks_collection:
    if task["assignee"]["user_id"] == "user_alice":
        task["assignee"]["display_name"] = "Alice Johnson"

# They FORGET to update the comments collection.
# (Easy to miss — comments is a separate collection with 100k+ documents)

print("=== Task Assignee ===")
print(f"  {tasks_collection[0]['assignee']['display_name']}")

print("\n=== Comments on Task 001 ===")
for comment in comments_collection:
    if comment["task_id"] == "task_001":
        print(f"  [{comment['author']['display_name']}]: {comment['body']}")
```

**Questions:**

1. **Predict the exact output.** Write every line, including the display names.

2. **Explain the inconsistency.** `user_id` is `"user_alice"` in all documents. Why does the same user appear under two different names after a single update?

3. **The Relational Alternative.** In PostgreSQL with a `users` table and `user_id` as a foreign key in both `tasks` and `comments`, a single `UPDATE users SET display_name = 'Alice Johnson' WHERE id = 'user_alice'` fixes all references simultaneously. What is the structural difference between a foreign key (reference) and an embedded dict (copy) that makes this possible in PostgreSQL but impossible in the embedded document model?

4. **The Design Judgment.** The lecture states: "Embed when data doesn't change independently." From this example, identify **one specific field** that *should* be embedded (an independent copy is correct behavior) and **one field** that *should* be referenced (always reflect current state). Justify each choice.

---

## Exercise 1.3: The Leaderboard That Overwrites

**Context:** A team builds a weekly task-completion leaderboard using Redis Sorted Sets. The following class faithfully reproduces the semantics of `ZADD`, `ZINCRBY`, `ZRANGE`, and `ZSCORE`. The comments describe the exact Redis command being simulated.

```python
import time

class SimulatedRedis:
    """
    Faithful simulation of Redis Sorted Set + String operations.
    Semantics match real Redis exactly.
    """

    def __init__(self):
        self._sorted_sets: dict[str, dict[str, float]] = {}
        self._strings: dict[str, str] = {}
        self._expiry: dict[str, float] = {}

    # --- Sorted Set commands ---

    def zadd(self, key: str, mapping: dict[str, float]) -> int:
        """
        Redis: ZADD key score member [score member ...]
        Sets the score for each member. If member already exists,
        its score is REPLACED (not added to).
        Returns: number of NEW members added (0 if member existed).
        """
        if key not in self._sorted_sets:
            self._sorted_sets[key] = {}
        new_count = 0
        for member, score in mapping.items():
            if member not in self._sorted_sets[key]:
                new_count += 1
            self._sorted_sets[key][member] = score  # SET, never ADD TO
        return new_count

    def zincrby(self, key: str, increment: float, member: str) -> float:
        """
        Redis: ZINCRBY key increment member
        Increments the score of member by increment.
        Creates member with score=increment if it doesn't exist.
        Returns: new score.
        """
        if key not in self._sorted_sets:
            self._sorted_sets[key] = {}
        current = self._sorted_sets[key].get(member, 0.0)
        self._sorted_sets[key][member] = current + increment
        return self._sorted_sets[key][member]

    def zrange(self, key: str, start: int, stop: int,
               rev: bool = False) -> list[str]:
        """Redis: ZRANGE key start stop [REV] — returns member names only."""
        if key not in self._sorted_sets:
            return []
        items = sorted(
            self._sorted_sets[key].items(),
            key=lambda x: x[1],
            reverse=rev,
        )
        if stop == -1:
            stop = len(items) - 1
        return [m for m, _ in items[start : stop + 1]]

    def zscore(self, key: str, member: str) -> float | None:
        """Redis: ZSCORE key member — returns the score or None."""
        return self._sorted_sets.get(key, {}).get(member)

    def zcard(self, key: str) -> int:
        """Redis: ZCARD key — returns number of members in sorted set."""
        return len(self._sorted_sets.get(key, {}))

    # --- String commands ---

    def set(self, key: str, value: str, ex: int | None = None) -> None:
        """Redis: SET key value [EX seconds]"""
        self._strings[key] = value
        if ex:
            self._expiry[key] = time.monotonic() + ex

    def get(self, key: str) -> str | None:
        """Redis: GET key — returns None if expired or not found."""
        if key in self._expiry and time.monotonic() > self._expiry[key]:
            self._strings.pop(key, None)
            self._expiry.pop(key, None)
            return None
        return self._strings.get(key)


r = SimulatedRedis()

# --- Monday: Scores from weekend catch-up work ---
r.zadd("leaderboard:week9", {"alice": 5, "bob": 3, "charlie": 7})
print(f"Monday  — Members in leaderboard: {r.zcard('leaderboard:week9')}")

# --- Wednesday: Performance report submitted ---
# Intention: "Charlie completed 6 more tasks this week."
new_entries = r.zadd("leaderboard:week9", {"charlie": 6})   # ← Student focus
print(f"Wednesday — New members added by ZADD: {new_entries}")
print(f"Wednesday — Charlie's score: {r.zscore('leaderboard:week9', 'charlie')}")

# --- Thursday: Alice closes a sprint ---
r.zincrby("leaderboard:week9", 8, "alice")
print(f"Thursday  — Alice's score: {r.zscore('leaderboard:week9', 'alice')}")

# --- Friday: Final rankings ---
print("\n=== End-of-Week Rankings (Highest First) ===")
for rank, member in enumerate(
    r.zrange("leaderboard:week9", 0, -1, rev=True), start=1
):
    score = r.zscore("leaderboard:week9", member)
    print(f"  #{rank}: {member} — {score:.0f} tasks")

# --- Cache scenario: session data with TTL ---
# (Note: sleep(2.1) below takes ~2 seconds intentionally.)
print("\n=== Session Cache ===")
r.set("session:alice", '{"role": "admin"}', ex=2)
print(f"Immediately:    {r.get('session:alice')}")
time.sleep(2.1)
print(f"After 2.1s:     {r.get('session:alice')}")
```

**Questions:**

1. **Predict the exact output** — every print statement, in order. Pay special attention to Wednesday's output.

2. **Explain the Wednesday result.** The developer intended to record that Charlie completed 6 *additional* tasks since Monday (he had 7). Does the code do that? What is Charlie's final score, and is it correct from a business perspective?

3. **Identify the correct operation.** What Redis command should the developer have used instead of `ZADD`? In one sentence each, state the semantic difference: what does `ZADD` do vs. what does `ZINCRBY` do?

4. **The PostgreSQL Equivalent.** What SQL statement is semantically equivalent to `ZINCRBY leaderboard:week9 8 alice`? Given that the SQL is arguably simpler to read, make the concrete case for using Redis for this leaderboard instead of PostgreSQL. Your argument must go beyond "Redis is faster in general."

---

## Level 1 Solutions

### Solution 1.1

**1. Correct Output:**
```
Products with warranty >= 1 year: 1
  - ThinkPad X1
```

**2. Mental Trace:**

| Product | Key Checked by `.get()` | Value Returned | Passes `>= 1`? |
|---|---|---|---|
| ThinkPad X1 | `"warranty_years"` | `3` (key exists) | ✅ Yes |
| MacBook Pro | `"warranty_years"` | `0` (key absent — default used) | ❌ No |
| Dell XPS 15 | `"warranty_years"` | `0` (key absent — default used) | ❌ No |

Python dict lookups are exact and case-sensitive. `"warrantyYears"` and `"warranty"` are entirely different keys from `"warranty_years"`. The values `2` and `1` are stored in those dicts — they just aren't findable by the query function.

**3. PostgreSQL Contrast:**

PostgreSQL prevents this with **column name enforcement at the DDL level**. After `ALTER TABLE products ADD COLUMN warranty_years INTEGER NOT NULL`, the column name `warranty_years` is the only valid column for that data. An `INSERT INTO products (warrantyYears, ...)` raises `psycopg2.errors.UndefinedColumn` immediately at write time, before any corrupt data reaches the table.

What you surrender: the database is no longer your last line of defence. In PostgreSQL, a typo in a migration fails loudly before data is touched. In a document store, bad data silently enters the collection and only reveals itself later as missing query results — the worst kind of bug.

**4. The Partial Fix — and Its Limits:**

The Pydantic fix helps significantly, but two scenarios still allow the problem:

- **Pre-existing data:** Documents inserted before the validation was added still have inconsistent field names. Pydantic only guards new writes. There is no equivalent of Alembic's `upgrade` to transform old documents.
- **Direct database access:** Any admin script, MongoDB shell, or migration tool that writes directly to the collection bypasses your FastAPI application and its Pydantic layer entirely. PostgreSQL's column constraint fires regardless of which connection or tool performs the insert.

The principle: **"schema-less" means "schema-in-application-code."** Pydantic is the correct mechanism — but it requires every write path to go through that layer, which the database cannot enforce for you.

---

### Solution 1.2

**1. Correct Output:**
```
=== Task Assignee ===
  Alice Johnson

=== Comments on Task 001 ===
  [Alice Smith]: I'll start with the endpoints.
  [Alice Smith]: Should we use UUID or integer PKs?
  [Bob Chen]: UUIDs — easier for distributed systems.
```

**2. The Inconsistency:**

The loop updates `tasks_collection` only. Each comment document has its own `author` dict — a separate, independent Python dict that was created and stored when the comment was inserted. Updating the task's `assignee["display_name"]` has zero effect on those dicts; they occupy different memory locations. The same `user_id` (`"user_alice"`) appears with two different names because embedding **duplicated** the name three times at insertion. There is no pointer between them after that moment.

**3. Reference vs. Copy:**

A PostgreSQL foreign key stores a *pointer* — the integer `user_id` in `tasks` and `comments` doesn't contain a name; it contains an address. `UPDATE users SET display_name = 'Alice Johnson'` changes the data at that address, and every row that holds the foreign key now "sees" the new value through that single address.

An embedded dict stores a *copy* — the name `"Alice Smith"` was copied into three separate dicts at insertion time. Those copies have no connection to each other or to the original user record. Updating one copy does not affect the others; there is no address to change, only independent values.

**4. Design Judgment:**

- **Should be embedded** — `"display_name"` *at the time of the comment*. Comments are historical records. If Alice changed her display name, old comments should arguably still show the name she used when she wrote them (the same reason order receipts embed the price paid, not the current price). This is an intentional snapshot.

- **Should be referenced** — `"user_id"`. This stable identifier is the anchor. If the application always wants to show the *current* display name (e.g., profile photos, role badges), it should look up `user_id` live. Embedding the current display name adds update complexity for no benefit.

The key question is always: "Is this a snapshot of the past or a reflection of the present?"

---

### Solution 1.3

**1. Correct Output:**
```
Monday  — Members in leaderboard: 3
Wednesday — New members added by ZADD: 0
Wednesday — Charlie's score: 6.0
Thursday  — Alice's score: 13.0

=== End-of-Week Rankings (Highest First) ===
  #1: alice — 13 tasks
  #2: charlie — 6 tasks
  #3: bob — 3 tasks

=== Session Cache ===
Immediately:    {"role": "admin"}
After 2.1s:     None
```

**2. The Wednesday Problem:**

`r.zadd("leaderboard:week9", {"charlie": 6})` **replaces** Charlie's score of 7 with 6. It does not add 6 to 7.

- `ZADD` returns `0` because charlie already exists (no new member added).
- Charlie's score drops from 7 to 6 — the developer recorded 6 completions and overwrote the evidence that Charlie ever had 7. The business report is now wrong.

**3. `ZADD` vs `ZINCRBY`:**

| Command | Semantics |
|---|---|
| `ZADD key score member` | **Sets** the score to an absolute value. Existing score is irrelevant and is discarded. |
| `ZINCRBY key increment member` | **Adds** `increment` to the current score atomically. Creates the member with `score = increment` if it does not exist. |

The correct command was `r.zincrby("leaderboard:week9", 6, "charlie")` → `7 + 6 = 13`.

**4. PostgreSQL Equivalent and Why Redis:**

The SQL equivalent is:
```sql
UPDATE leaderboard SET score = score + 8 WHERE member = 'alice' AND week = 'week9';
```

Redis wins here for three specific reasons:

- **Latency:** The leaderboard widget is re-read on every dashboard page load. Redis serves a `ZRANGE` in ~0.1ms; a PostgreSQL `SELECT ... ORDER BY score DESC LIMIT 10` involves query planning, potential disk reads, and connection overhead — typically 1–5ms even with an index. At 10,000 concurrent dashboard views, that difference matters.
- **Atomic increments under concurrency:** If 50 users complete tasks simultaneously, 50 `ZINCRBY` calls are each individually atomic (Redis is single-threaded). The PostgreSQL equivalent requires an explicit transaction to prevent a race condition on the `score + 8` read-modify-write.
- **Correct separation of concerns:** The leaderboard is a *computed aggregate* of task records, not the source of truth. PostgreSQL owns the task records; Redis owns the derived leaderboard. If Redis restarts, the leaderboard is rebuilt from PostgreSQL — source of truth is never at risk.

---

# LEVEL 2: DEBUG, FIX & VERIFY

## Exercise 2.1: The JSONB Sort That Lies

**Context:** A developer chose PostgreSQL + JSONB to solve the product catalog problem from Part 1 of the lecture — the right architectural decision. But a subtle bug in the query logic makes the search results silently wrong. No exception is raised. Product managers are filing tickets: "The search is broken — cheap, tiny products keep appearing at the top."

```python
# This code faithfully simulates the behaviour of a PostgreSQL JSONB query
# that forgets to cast the attribute value before comparing.
#
# In PostgreSQL:
#   attributes->>'screen_size'          returns TEXT, not FLOAT
#   TEXT > TEXT  is lexicographic        "6.7" > "13.0" evaluates to True
#   (attributes->>'screen_size')::float  would be correct — but is missing
#
# The Python below reproduces this exact failure mode.

products = [
    {"name": 'Monitor 27"',   "attributes": {"screen_size": 27.0}},
    {"name": "Laptop Pro 15", "attributes": {"screen_size": 15.6}},
    {"name": "Laptop Air 13", "attributes": {"screen_size": 13.3}},
    {"name": "Tablet 10",     "attributes": {"screen_size": 10.5}},
    {"name": "Phone 6.7",     "attributes": {"screen_size": 6.7}},
]

def search_by_min_screen(min_inches: float) -> list[dict]:
    """
    Return products with screen_size > min_inches, largest first.

    Simulates this PostgreSQL query (which contains the bug):

        SELECT name FROM products
        WHERE   attributes->>'screen_size' > '{min_inches}'
        ORDER BY attributes->>'screen_size' DESC;
    """
    filtered = [
        p for p in products
        if str(p["attributes"]["screen_size"]) > str(min_inches)   # Bug
    ]
    filtered.sort(
        key=lambda p: str(p["attributes"]["screen_size"]),          # Bug
        reverse=True,
    )
    return filtered

results = search_by_min_screen(13.0)

print(f"Screens larger than 13 inches ({len(results)} found):")
for p in results:
    print(f"  {p['name']}: {p['attributes']['screen_size']}\"")
```

---

**Task 2a: Identify the Flaw**

Without fixing anything:

1. Which product appears **first** in the results, and why is that a surprise?
2. Which product **should** be filtered out entirely but appears in the results?
3. Which product **should** appear first but doesn't?

---

**Task 2b: Explain the Mechanism**

1. Python compares `"6.7" > "13.0"`. Walk through the character-by-character comparison and state the result. Why does this contradict the numeric result?
2. Show the **correct numeric sort order** (descending) alongside the **actual lexicographic sort order** (descending) for all values that pass the filter. Present them side-by-side.
3. Name the specific PostgreSQL cast operator that would fix this query, and write the corrected SQL `WHERE` clause.

---

**Task 2c: Instrument to Prove Diagnosis**

Add print statements to expose exactly what comparisons are happening. Do **not** fix the logic — only add instrumentation:

```python
def search_by_min_screen_debug(min_inches: float) -> list[dict]:
    print(f"\n--- Filter comparisons (threshold: '{str(min_inches)}') ---")
    for p in products:
        screen_str = str(p["attributes"]["screen_size"])
        threshold_str = str(min_inches)
        passes = screen_str > threshold_str
        # Add your line here that prints the comparison clearly
        ???
    # ... rest unchanged
```

Show the output that proves exactly why `Phone 6.7` passes the filter.

---

**Task 2d: Fix the Code**

Fix `search_by_min_screen` so it performs correct numeric comparison and sorts numerically. Include the corrected PostgreSQL query as a comment inside the function.

---

**Task 2e: Write Verification Tests**

Write two tests. Both must **fail** on the original broken function and **pass** on the fixed version:

```python
def test_phone_excluded_from_large_screen_results():
    """Phone 6.7 must NOT appear when searching for screens > 13 inches."""
    results = search_by_min_screen(13.0)
    result_names = [p["name"] for p in results]
    assert ???

def test_monitor_appears_first():
    """The 27-inch monitor must be the top result when searching > 13 inches."""
    results = search_by_min_screen(13.0)
    assert len(results) > 0
    assert ???
```

---

## Exercise 2.2: The Ghost Assignee

**Context:** A developer built a task management system using a document-store design. All CRUD tests pass. But after a "user account deletion" feature was shipped, a burst of crash reports arrived from the dashboard. The exception is raised silently in production — `except Exception` was catching it until a logging update revealed the true cause.

```python
class DocumentCollection:
    """Simulates a MongoDB collection. No foreign key constraints whatsoever."""

    def __init__(self, name: str):
        self.name = name
        self._docs: dict[str, dict] = {}

    def insert_one(self, doc_id: str, doc: dict) -> None:
        self._docs[doc_id] = {**doc, "_id": doc_id}

    def find_one(self, doc_id: str) -> dict | None:
        return self._docs.get(doc_id)

    def find_many(self, predicate) -> list[dict]:
        return [d for d in self._docs.values() if predicate(d)]

    def delete_one(self, doc_id: str) -> bool:
        return bool(self._docs.pop(doc_id, None))

    def count(self) -> int:
        return len(self._docs)


users = DocumentCollection("users")
tasks = DocumentCollection("tasks")

# Setup
users.insert_one("u_001", {"name": "Alice",   "email": "alice@corp.com",  "role": "engineer"})
users.insert_one("u_002", {"name": "Bob",     "email": "bob@corp.com",    "role": "engineer"})
users.insert_one("u_003", {"name": "Charlie", "email": "charlie@corp.com","role": "manager"})

tasks.insert_one("t_001", {"title": "Build login API",  "assignee_id": "u_001", "status": "open"})
tasks.insert_one("t_002", {"title": "Write unit tests", "assignee_id": "u_001", "status": "open"})
tasks.insert_one("t_003", {"title": "Deploy staging",   "assignee_id": "u_002", "status": "open"})
tasks.insert_one("t_004", {"title": "Approve PR",       "assignee_id": "u_003", "status": "open"})

def get_open_tasks_with_assignees() -> list[dict]:
    """Return open tasks with full assignee details."""
    open_tasks = tasks.find_many(lambda t: t["status"] == "open")
    result = []
    for task in open_tasks:
        assignee = users.find_one(task["assignee_id"])
        result.append({
            "title": task["title"],
            "assignee_name": assignee["name"],     # ← What if assignee is None?
            "assignee_email": assignee["email"],   # ←
        })
    return result

def offboard_user(user_id: str) -> None:
    """Remove a terminated employee from the system."""
    users.delete_one(user_id)
    # Note: task reassignment is NOT implemented

# HR terminates Alice
offboard_user("u_001")

print(f"Users remaining: {users.count()}")
print(f"Tasks remaining: {tasks.count()}")

try:
    dashboard = get_open_tasks_with_assignees()
    for t in dashboard:
        print(f"  [{t['title']}] → {t['assignee_name']}")
except Exception as e:
    print(f"Dashboard crashed: {type(e).__name__}: {e}")
```

---

**Task 2a: Identify the Flaw**

1. What exception is raised when `get_open_tasks_with_assignees()` runs after Alice is offboarded? Name it precisely.
2. Which specific line raises it?
3. Name the exact PostgreSQL feature that would have made this scenario architecturally impossible, and state what it would have done automatically when `offboard_user` deleted Alice's row.

---

**Task 2b: Explain the Mechanism**

1. After `offboard_user("u_001")` completes, what do `t_001` and `t_002` still contain in their `assignee_id` field? Has anything changed in those documents?
2. What does `users.find_one("u_001")` return after the deletion?
3. The code executes `assignee["name"]` on that return value. Trace what Python does with this operation, step by step, and state the error.

---

**Task 2c: Instrument to Prove Diagnosis**

Without fixing `offboard_user`, add an **orphan detection function** and call it after the offboarding:

```python
def find_orphaned_tasks() -> list[dict]:
    """Return all tasks whose assignee_id points to a non-existent user."""
    # Your implementation here — use tasks.find_many() and users.find_one()
    pass

# Call it here and print the results
```

Show the output that exposes the dangling references.

---

**Task 2d: Fix the Code**

Apply two independent fixes:

**Fix 1 — Cascade in `offboard_user`:** When a user is deleted, set `assignee_id = None` on all their tasks. The signature `offboard_user(user_id: str) -> None` must not change.

**Fix 2 — Defensive guard in `get_open_tasks_with_assignees`:** Handle `None` gracefully. Unassigned tasks should return `"assignee_name": "Unassigned"`.

---

**Task 2e: Write Verification Tests**

```python
def test_offboarding_nullifies_task_assignments():
    """After offboarding, none of the user's tasks should reference the deleted user."""
    # Hint: call find_orphaned_tasks() after offboard_user() and assert it returns []
    ???

def test_dashboard_handles_unassigned_tasks_without_crashing():
    """get_open_tasks_with_assignees must not raise when a task has assignee_id = None."""
    ???
```

---

## Level 2 Solutions

### Solution 2.1

**Task 2a: Identify the Flaw**

1. **`Phone 6.7`** appears first — surprising because 6.7 inches is far less than 13, so it should have been filtered out before sorting.
2. **`Phone 6.7`** should be filtered out (6.7 < 13.0 numerically) but passes the filter because `"6.7" > "13.0"` is `True` as a string comparison.
3. **`Monitor 27"`** should appear first (the largest screen) but is pushed to second place behind `Phone 6.7`.

**Task 2b: Explain the Mechanism**

**`"6.7" > "13.0"` character by character:**
1. Compare first characters: `"6"` vs `"1"`
2. `ord("6")` = 54, `ord("1")` = 49 → 54 > 49 → **True**
3. Comparison stops here. The rest of the string is irrelevant.

This is lexicographic (dictionary) ordering. Just as `"banana"` < `"cherry"` because `b` < `c`, `"13.0"` < `"6.7"` because `1` < `6`.

**Sort order comparison:**

| Numeric (correct, descending) | Lexicographic (actual, descending) |
|---|---|
| 27.0 — 1st | 6.7 — 1st (`"6"` > `"2"`) |
| 15.6 — 2nd | 27.0 — 2nd (`"2"` > `"1"`) |
| 13.3 — 3rd | 15.6 — 3rd (`"15"` > `"13"`) |
| ~~6.7 — filtered out~~ | 13.3 — 4th |

**Corrected PostgreSQL query:**
```sql
-- Bug (text comparison):
WHERE attributes->>'screen_size' > '13.0'

-- Fix (cast to float before comparison):
WHERE (attributes->>'screen_size')::float > 13.0
ORDER BY (attributes->>'screen_size')::float DESC;
```

**Task 2c: Instrument**

```python
def search_by_min_screen_debug(min_inches: float) -> list[dict]:
    print(f"\n--- Filter comparisons (threshold: '{str(min_inches)}') ---")
    for p in products:
        screen_str = str(p["attributes"]["screen_size"])
        threshold_str = str(min_inches)
        passes = screen_str > threshold_str
        print(f"  '{screen_str}' > '{threshold_str}' → {passes}  ({p['name']})")
```

Output:
```
--- Filter comparisons (threshold: '13.0') ---
  '27.0' > '13.0' → True   (Monitor 27")
  '15.6' > '13.0' → True   (Laptop Pro 15)
  '13.3' > '13.0' → True   (Laptop Air 13)
  '10.5' > '13.0' → False  (Tablet 10)     ← correctly excluded
  '6.7'  > '13.0' → True   (Phone 6.7)     ← BUG: incorrectly included
```

**Task 2d: Fix the Code**

```python
def search_by_min_screen(min_inches: float) -> list[dict]:
    """
    Corrected PostgreSQL query equivalent:

        SELECT name FROM products
        WHERE   (attributes->>'screen_size')::float > %(min_inches)s
        ORDER BY (attributes->>'screen_size')::float DESC;

    The ::float cast converts the TEXT returned by ->> into a FLOAT
    before the comparison. Without the cast, > is lexicographic.
    """
    filtered = [
        p for p in products
        if float(p["attributes"]["screen_size"]) > float(min_inches)  # Fixed
    ]
    filtered.sort(
        key=lambda p: float(p["attributes"]["screen_size"]),           # Fixed
        reverse=True,
    )
    return filtered
```

**Task 2e: Verification Tests**

```python
def test_phone_excluded_from_large_screen_results():
    results = search_by_min_screen(13.0)
    result_names = [p["name"] for p in results]
    assert "Phone 6.7" not in result_names, (
        f"Phone 6.7 should be excluded but appeared in: {result_names}"
    )

def test_monitor_appears_first():
    results = search_by_min_screen(13.0)
    assert len(results) > 0, "Should have results"
    assert results[0]["name"] == 'Monitor 27"', (
        f"Expected Monitor 27 first, got: {results[0]['name']}"
    )
```

---

### Solution 2.2

**Task 2a: Identify the Flaw**

1. **`TypeError: 'NoneType' object is not subscriptable`**
2. The line `"assignee_name": assignee["name"]` — `assignee` is `None` because `users.find_one("u_001")` returns `None` after the user was deleted.
3. In PostgreSQL: `assignee_id INTEGER REFERENCES users(id) ON DELETE SET NULL` — deleting Alice's row would automatically set `assignee_id = NULL` in every task that referenced her. The database enforces this regardless of which code path performs the delete.

**Task 2b: Explain the Mechanism**

1. After `offboard_user("u_001")`, tasks `t_001` and `t_002` are completely **unchanged**. Their `assignee_id` field still holds `"u_001"`. The document store has no mechanism to notify or update any other collection when a document is deleted.
2. `users.find_one("u_001")` returns `None` — the key no longer exists in `_docs`.
3. Python attempts `None["name"]` → `None` is not a subscriptable object → `TypeError: 'NoneType' object is not subscriptable`.

**Task 2c: Instrument**

```python
def find_orphaned_tasks() -> list[dict]:
    all_tasks = tasks.find_many(lambda t: True)
    return [
        t for t in all_tasks
        if t.get("assignee_id") and users.find_one(t["assignee_id"]) is None
    ]

orphaned = find_orphaned_tasks()
print(f"\nOrphaned tasks after offboarding: {len(orphaned)}")
for t in orphaned:
    print(f"  [{t['title']}] → assignee_id: {t['assignee_id']} (USER NOT FOUND)")
```

Output:
```
Orphaned tasks after offboarding: 2
  [Build login API] → assignee_id: u_001 (USER NOT FOUND)
  [Write unit tests] → assignee_id: u_001 (USER NOT FOUND)
```

**Task 2d: Fix the Code**

```python
def offboard_user(user_id: str) -> None:
    """Remove a user and manually cascade to their task assignments."""
    users.delete_one(user_id)

    # Manual cascade: replicate what PostgreSQL ON DELETE SET NULL does automatically.
    # In a real MongoDB implementation this would be:
    #   db.tasks.update_many({"assignee_id": user_id}, {"$set": {"assignee_id": None}})
    for task in tasks.find_many(lambda t: t.get("assignee_id") == user_id):
        task["assignee_id"] = None

def get_open_tasks_with_assignees() -> list[dict]:
    """Return open tasks. Handles None assignees gracefully (defense in depth)."""
    open_tasks = tasks.find_many(lambda t: t["status"] == "open")
    result = []
    for task in open_tasks:
        assignee = (
            users.find_one(task["assignee_id"])
            if task.get("assignee_id")
            else None
        )
        result.append({
            "title": task["title"],
            "assignee_name": assignee["name"] if assignee else "Unassigned",
            "assignee_email": assignee["email"] if assignee else None,
        })
    return result
```

**Task 2e: Verification Tests**

```python
def test_offboarding_nullifies_task_assignments():
    """After offboarding, the user's tasks must have assignee_id = None."""
    offboard_user("u_001")
    orphaned = find_orphaned_tasks()
    assert len(orphaned) == 0, (
        f"Found {len(orphaned)} orphaned tasks after offboarding"
    )

def test_dashboard_handles_unassigned_tasks_without_crashing():
    """Dashboard must not raise when tasks have no assignee."""
    offboard_user("u_001")  # Creates tasks with assignee_id = None
    try:
        dashboard = get_open_tasks_with_assignees()
        for task in dashboard:
            assert "assignee_name" in task
            # Tasks that had Alice as assignee must say "Unassigned", not crash
        print("Test passed: no crash on unassigned tasks.")
    except (TypeError, AttributeError) as e:
        assert False, f"Dashboard crashed with: {e}"
```

---

# LEVEL 2.5: EXTEND EXISTING CODE

**Context:** The following product search system is working and correct. It uses JSONB-style flexible attribute storage — the same pattern the lecture introduces in Part 1.3. Each extension adds a new capability. The **existing test suite must continue to pass after every extension without modification.**

```python
from typing import Any

products_db: list[dict] = [
    {
        "id": 1, "name": "ThinkPad X1", "price": 1299.99, "category": "electronics",
        "attributes": {
            "screen_size": 14.0,
            "connectivity": ["wifi", "bluetooth", "usb-c"],
            "warranty_years": 3,
        },
    },
    {
        "id": 2, "name": "MacBook Pro 16", "price": 2499.99, "category": "electronics",
        "attributes": {
            "screen_size": 16.1,
            "connectivity": ["wifi", "bluetooth", "thunderbolt"],
            "warranty_years": 1,
        },
    },
    {
        "id": 3, "name": "iPad Air", "price": 749.99, "category": "electronics",
        "attributes": {
            "screen_size": 10.9,
            "connectivity": ["wifi", "bluetooth", "usb-c"],
            "warranty_years": 1,
        },
    },
    {
        "id": 4, "name": "Python Book", "price": 45.00, "category": "books",
        "attributes": {
            "isbn": "978-0134853987",
            "pages": 432,
            "author": "Luciano Ramalho",
        },
    },
    {
        "id": 5, "name": "Gaming Chair", "price": 399.99, "category": "furniture",
        "attributes": {
            "weight_kg": 18.5,
            "assembly_required": True,
            "room_type": ["office", "gaming"],
        },
    },
]


def search_products(
    category: str | None = None,
    min_screen_size: float | None = None,
) -> list[dict]:
    """Search products by category and/or minimum screen size. Sorted by price ascending."""
    results = products_db.copy()

    if category:
        results = [p for p in results if p["category"] == category]

    if min_screen_size is not None:
        results = [
            p for p in results
            if float(p["attributes"].get("screen_size", 0)) >= min_screen_size
        ]

    results.sort(key=lambda p: p["price"])
    return results


# ── EXISTING TEST SUITE — must remain green after every extension ──

def test_category_filter():
    results = search_products(category="electronics")
    assert len(results) == 3
    assert all(p["category"] == "electronics" for p in results)

def test_screen_size_filter():
    results = search_products(category="electronics", min_screen_size=14.0)
    assert len(results) == 2
    assert all(p["attributes"]["screen_size"] >= 14.0 for p in results)

def test_sort_by_price_ascending():
    results = search_products(category="electronics")
    prices = [p["price"] for p in results]
    assert prices == sorted(prices)

test_category_filter()
test_screen_size_filter()
test_sort_by_price_ascending()
print("Existing tests pass ✅")
```

---

### Extension 1: Price Range Filter

Add `min_price: float | None = None` and `max_price: float | None = None` parameters to `search_products`. Both are optional and inclusive. Existing calls with no price arguments must behave identically.

**New tests to pass:**
```python
def test_price_range_mid_tier():
    results = search_products(category="electronics", min_price=700.0, max_price=1500.0)
    assert len(results) == 2
    names = [p["name"] for p in results]
    assert "ThinkPad X1" in names
    assert "iPad Air" in names
    assert "MacBook Pro 16" not in names   # $2499 is over the max

def test_price_range_no_results():
    results = search_products(min_price=10_000.0)
    assert len(results) == 0

def test_original_filters_still_work():
    """Existing behaviour: category + screen_size combination unchanged."""
    results = search_products(category="electronics", min_screen_size=14.0)
    assert len(results) == 2
```

---

### Extension 2: Array Attribute Search

Add `required_connectivity: list[str] | None = None`. When provided, only products whose `connectivity` attribute contains **all** of the specified values are returned. Products without a `connectivity` key (books, furniture) must be silently excluded when this filter is active.

**New tests to pass:**
```python
def test_bluetooth_only():
    results = search_products(category="electronics", required_connectivity=["bluetooth"])
    assert len(results) == 3   # all electronics have bluetooth

def test_usb_c_excludes_thunderbolt_mac():
    results = search_products(required_connectivity=["usb-c"])
    assert len(results) == 2
    names = [p["name"] for p in results]
    assert "MacBook Pro 16" not in names   # has thunderbolt, not usb-c

def test_multiple_requirements_are_AND_not_OR():
    """Product must have BOTH wifi AND usb-c — not either one."""
    results = search_products(required_connectivity=["wifi", "usb-c"])
    assert len(results) == 2   # ThinkPad and iPad; MacBook has wifi but not usb-c

def test_non_electronics_excluded_from_connectivity_filter():
    results = search_products(required_connectivity=["bluetooth"])
    assert all(p["category"] == "electronics" for p in results)
```

---

### Extension 3: Cache-Aside Pattern

Add a cache layer using a Python dict as a simulated Redis store and implement the **cache-aside pattern**:

1. On each call (when `use_cache=True`), compute a deterministic cache key from all parameters.
2. Return the cached result immediately on a hit.
3. On a miss, compute the result, store it, and return it.

**Requirements:**
- `use_cache: bool = True` added as a parameter (existing tests should pass with `use_cache=False` to avoid inter-test state)
- Different queries must produce different cache keys — `category="electronics"` and `category="books"` must not share an entry
- A standalone `invalidate_cache()` function clears all entries

**New tests to pass:**
```python
def test_cache_hit_returns_same_result():
    invalidate_cache()
    first  = search_products(category="electronics", use_cache=True)
    second = search_products(category="electronics", use_cache=True)  # hit
    assert first == second

def test_different_queries_have_different_cache_entries():
    invalidate_cache()
    elec  = search_products(category="electronics", use_cache=True)
    books = search_products(category="books", use_cache=True)
    assert elec != books
    assert any(p["category"] == "books" for p in books)

def test_invalidation_does_not_corrupt_results():
    invalidate_cache()
    before = search_products(category="electronics", use_cache=True)
    invalidate_cache()
    after  = search_products(category="electronics", use_cache=True)  # fresh fetch
    assert before == after
```

---

## Level 2.5 Solutions

### Extension 1 Solution

```python
def search_products(
    category: str | None = None,
    min_screen_size: float | None = None,
    min_price: float | None = None,   # NEW
    max_price: float | None = None,   # NEW
) -> list[dict]:
    results = products_db.copy()

    if category:
        results = [p for p in results if p["category"] == category]
    if min_screen_size is not None:
        results = [
            p for p in results
            if float(p["attributes"].get("screen_size", 0)) >= min_screen_size
        ]
    if min_price is not None:
        results = [p for p in results if p["price"] >= min_price]
    if max_price is not None:
        results = [p for p in results if p["price"] <= max_price]

    results.sort(key=lambda p: p["price"])
    return results
```

**Why:** Each filter is an independent `None`-guarded step. Defaulting to `None` means no filter is applied when the argument is omitted, preserving all existing call sites. The sort always happens last, after all filters have narrowed the set.

---

### Extension 2 Solution

```python
def search_products(
    category: str | None = None,
    min_screen_size: float | None = None,
    min_price: float | None = None,
    max_price: float | None = None,
    required_connectivity: list[str] | None = None,   # NEW
) -> list[dict]:
    results = products_db.copy()

    if category:
        results = [p for p in results if p["category"] == category]
    if min_screen_size is not None:
        results = [
            p for p in results
            if float(p["attributes"].get("screen_size", 0)) >= min_screen_size
        ]
    if min_price is not None:
        results = [p for p in results if p["price"] >= min_price]
    if max_price is not None:
        results = [p for p in results if p["price"] <= max_price]
    if required_connectivity:
        results = [
            p for p in results
            if "connectivity" in p["attributes"]               # Silently skip books/furniture
            and all(
                req in p["attributes"]["connectivity"]
                for req in required_connectivity
            )
        ]

    results.sort(key=lambda p: p["price"])
    return results
```

**Why the `all(...)` pattern:** It evaluates to `True` only if **every** required type is present in the product's list. This is AND semantics, not OR. The guard `"connectivity" in p["attributes"]` mirrors the behaviour of `attributes ? 'connectivity'` in PostgreSQL JSONB — the existence-check operator returns `False` rather than raising an error when the key is absent, silently excluding books and furniture.

---

### Extension 3 Solution

```python
_cache: dict[str, list[dict]] = {}

def invalidate_cache() -> None:
    """Clear all cached results. Redis equivalent: DEL search:* or FLUSHDB."""
    _cache.clear()

def _make_cache_key(
    category: str | None,
    min_screen_size: float | None,
    min_price: float | None,
    max_price: float | None,
    required_connectivity: list[str] | None,
) -> str:
    # Sort connectivity so ["usb-c", "wifi"] and ["wifi", "usb-c"] produce the same key.
    sorted_conn = sorted(required_connectivity) if required_connectivity else None
    return f"search:{category}:{min_screen_size}:{min_price}:{max_price}:{sorted_conn}"


def search_products(
    category: str | None = None,
    min_screen_size: float | None = None,
    min_price: float | None = None,
    max_price: float | None = None,
    required_connectivity: list[str] | None = None,
    use_cache: bool = True,                          # NEW
) -> list[dict]:

    # ── Cache-Aside Pattern ──
    # Step 1: Check cache (if enabled)
    if use_cache:
        key = _make_cache_key(
            category, min_screen_size, min_price, max_price, required_connectivity
        )
        if key in _cache:
            return _cache[key]   # Cache HIT

    # Step 2: Cache MISS — compute result
    results = products_db.copy()

    if category:
        results = [p for p in results if p["category"] == category]
    if min_screen_size is not None:
        results = [
            p for p in results
            if float(p["attributes"].get("screen_size", 0)) >= min_screen_size
        ]
    if min_price is not None:
        results = [p for p in results if p["price"] >= min_price]
    if max_price is not None:
        results = [p for p in results if p["price"] <= max_price]
    if required_connectivity:
        results = [
            p for p in results
            if "connectivity" in p["attributes"]
            and all(req in p["attributes"]["connectivity"] for req in required_connectivity)
        ]

    results.sort(key=lambda p: p["price"])

    # Step 3: Store before returning
    if use_cache:
        _cache[key] = results

    return results
```

**Why the connectivity sort matters:** If `["usb-c", "wifi"]` and `["wifi", "usb-c"]` generated different keys, the same logical query would create two separate cache entries. Sorting before building the key makes it idempotent — exactly the same reason you should normalise input before building a Redis key in production. The `use_cache=False` path in the existing tests ensures each test fetches fresh data without depending on cache state left by a previous test — the same isolation principle as rolling back a SQLAlchemy session after each test.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## The Scenario: EventTrack SaaS

You are designing the database layer for **EventTrack**, a SaaS platform for conference organizers. The system has four functional areas:

1. **Organizations, Events, Sessions, and Registrations** — standard relational entities with clear parent-child relationships.
2. **Analytics Events** — every user action (view, click, registration, survey response) is logged. Different action types carry different payload shapes. New action types are added every two-week sprint.
3. **Speaker Leaderboard** — a running engagement score per speaker, updated on every relevant attendee action. Displayed as a live widget on every attendee's device, refreshed every 5 seconds.
4. **Billing** — each organization prepays a "session credit" balance. Every attendee registration must atomically consume exactly 1 credit.

**Hard Constraints:**

| Label | Constraint |
|---|---|
| **A — Atomic Billing** | Credit deduction and attendee registration must succeed together or fail together. If the organization has 0 credits, neither the credit nor the registration is saved. |
| **B — Fast Leaderboard** | `get_speaker_leaderboard` must complete in under 1ms. It must not perform a full scan of the analytics event log. |
| **C — Schema-Free Analytics** | A brand-new analytics event type must be insertable without any database migration (no `ALTER TABLE`, no new collection definition). |
| **D — Max 2 Databases** | To limit operational complexity you may use at most 2 different database technologies. |

---

**Provided Test Suite — Your Implementation Must Pass All of These:**

```python
def test_constraint_A_atomic_registration(system):
    """Constraint A: Registration and credit deduction are atomic."""
    assert system.get_credit_balance("org_empty") == 0

    try:
        system.register_attendee(
            session_id="sess_001",
            user_id="user_001",
            organizer_id="org_empty",
            ticket_type="standard",
            price_paid=99.0,
        )
        assert False, "Should have raised — org has 0 credits"
    except Exception:
        pass  # Expected

    # Critical: balance must be unchanged (registration was rolled back)
    assert system.get_credit_balance("org_empty") == 0


def test_constraint_B_leaderboard_ordered(system):
    """Constraint B: Leaderboard results are sorted by score descending."""
    results = system.get_speaker_leaderboard("event_001", top_n=3)
    assert len(results) <= 3
    if len(results) > 1:
        scores = [r["score"] for r in results]
        assert scores == sorted(scores, reverse=True), (
            "Leaderboard must be highest score first"
        )


def test_constraint_C_new_event_type_requires_no_migration(system):
    """Constraint C: A never-before-seen event type must be insertable without error."""
    system.log_analytics_event(
        event_type="sponsor_scan",          # Brand-new type — never existed before
        user_id="user_001",
        payload={
            "sponsor_id": "sp_001",
            "qr_location": "booth_14",
            "scan_device": "ios",
        },
    )
    # If we reach this line without exception, Constraint C is met.


def test_leaderboard_updates_on_session_view(system):
    """Leaderboard score increments when a session_view event is logged."""
    system.log_analytics_event(
        event_type="session_view",
        user_id="attendee_001",
        payload={"speaker_id": "speaker_A", "duration_seconds": 120},
    )
    system.log_analytics_event(
        event_type="session_view",
        user_id="attendee_002",
        payload={"speaker_id": "speaker_A", "duration_seconds": 90},
    )
    leaderboard = system.get_speaker_leaderboard("event_001")
    scores = {entry["speaker_id"]: entry["score"] for entry in leaderboard}
    assert scores.get("speaker_A", 0) == 2


def test_duplicate_event_not_double_counted(system):
    """An event with the same event_id must only affect the leaderboard once."""
    for _ in range(2):   # Log the same event twice (network retry simulation)
        system.log_analytics_event(
            event_type="session_view",
            user_id="attendee_001",
            payload={"event_id": "evt_dedupe_001", "speaker_id": "speaker_B",
                     "duration_seconds": 60},
        )
    leaderboard = system.get_speaker_leaderboard("event_001")
    scores = {e["speaker_id"]: e["score"] for e in leaderboard}
    assert scores.get("speaker_B", 0) == 1, (
        f"Duplicate was double-counted: score is {scores.get('speaker_B')}"
    )
```

---

### Task 3a: Architecture Justification

Write an Architecture Decision Record (as a docstring or comments block) that:

1. Names the 2 databases you chose.
2. States which data lives in each one.
3. Maps each constraint (A, B, C, D) to the specific database feature that satisfies it. Be precise: name the mechanism, not just the product.

**The anti-pattern to avoid:** "Use MongoDB for everything because it's flexible." Explain specifically why this fails Constraint A and complicates Constraint B.

---

### Task 3b: Implement the Core System

Using the `SimulatedRedis` from Exercise 1.3 (with `zincrby`, `zrange`, `sadd`, `sismember`) and Python dicts to simulate PostgreSQL, implement an `EventTrackSystem` class with:

- `register_attendee(session_id, user_id, organizer_id, ticket_type, price_paid) → dict`
- `log_analytics_event(event_type, user_id, payload) → None`
- `get_speaker_leaderboard(event_id, top_n=10) → list[dict]`
- `get_credit_balance(organizer_id) → int`

The class should initialise with `"org_empty"` having 0 credits and `"org_001"` having 100 credits.

---

### Task 3c: Idempotent Leaderboard Updates

Extend `log_analytics_event` so that:
- When `event_type == "session_view"` and `"speaker_id"` is in the payload, the speaker's score in `leaderboard:event_001` is incremented by 1.
- If the same `event_id` is logged twice, the leaderboard is only incremented once (idempotency).
- If no `event_id` is provided in the payload, a unique ID is generated to prevent all events from colliding with each other.

Pass `test_duplicate_event_not_double_counted` and `test_leaderboard_updates_on_session_view`.

---

## Level 3 Solutions

### Solution 3a: Architecture Decision Record

```python
"""
EVENTTRACK — ARCHITECTURE DECISION RECORD
==========================================

CHOSEN DATABASES:  PostgreSQL  +  Redis

─── DATA ALLOCATION ───────────────────────────────────────────────

  PostgreSQL:
    - organizations, organizer credit balances
    - events, sessions, registrations
    - analytics event log (envelope + JSONB payload column)

  Redis:
    - speaker leaderboard (Sorted Sets)
    - processed event IDs (Sets — for idempotency deduplication)

─── CONSTRAINT MAPPING ────────────────────────────────────────────

  Constraint A (Atomic Billing):
    Database:   PostgreSQL
    Mechanism:  ACID TRANSACTION (BEGIN / COMMIT / ROLLBACK)
    Detail:     The credit deduction (UPDATE organizers SET credits = credits - 1)
                and the registration INSERT execute inside a single transaction.
                If credits = 0, the UPDATE touches 0 rows; the transaction
                detects this and rolls back before the INSERT is reached.
                MongoDB multi-document transactions exist (since v4.0) but are
                slower and less mature — and we already have PostgreSQL for the
                naturally relational billing data.

  Constraint B (Fast Leaderboard < 1ms):
    Database:   Redis
    Mechanism:  SORTED SET — ZINCRBY + ZRANGE
    Detail:     ZRANGE on a Sorted Set is O(log N + M) where M is the result
                count. For a top-10 leaderboard it is effectively O(1).
                A PostgreSQL GROUP BY + aggregate over a growing analytics table
                would be O(E) where E = total events — unbounded.
                The leaderboard stores a PRE-COMPUTED result, not raw data.
                PostgreSQL remains the source of truth; Redis holds the derived view.

  Constraint C (Schema-Free Analytics):
    Database:   PostgreSQL with JSONB payload column
    Mechanism:  JSONB flexible column
    Detail:     Each analytics event has a fixed envelope:
                  (event_id, event_type, user_id, logged_at)
                and a JSONB `payload` column for type-specific data.
                New event types = new payload shapes, zero ALTER TABLE.
                Using MongoDB for analytics alone would require a third database
                (violating Constraint D) or replacing PostgreSQL entirely
                (losing the ACID transactions needed for Constraint A).

  Constraint D (Max 2 Databases):
    PostgreSQL + Redis = exactly 2. ✅

─── WHY NOT MONGODB FOR EVERYTHING ────────────────────────────────
  MongoDB fails Constraint A: multi-document transactions in MongoDB
  are slower and require replica set configuration. The registration
  and credit tables have a clear relational structure with strict
  ACID requirements — this is precisely where PostgreSQL excels.

  MongoDB complicates Constraint B: you would still need Redis for the
  leaderboard. Adding MongoDB on top would exceed the Constraint D limit.
"""
```

---

### Solution 3b & 3c: Full Implementation

```python
import uuid
import time

class SimulatedRedis:
    """Extended Redis simulation: Sorted Sets + Sets for idempotency."""

    def __init__(self):
        self._sorted_sets: dict[str, dict[str, float]] = {}
        self._sets: dict[str, set[str]] = {}

    def zincrby(self, key: str, increment: float, member: str) -> float:
        if key not in self._sorted_sets:
            self._sorted_sets[key] = {}
        current = self._sorted_sets[key].get(member, 0.0)
        self._sorted_sets[key][member] = current + increment
        return self._sorted_sets[key][member]

    def zrange(self, key: str, start: int, stop: int,
               rev: bool = False) -> list[tuple[str, float]]:
        if key not in self._sorted_sets:
            return []
        items = sorted(
            self._sorted_sets[key].items(), key=lambda x: x[1], reverse=rev
        )
        if stop == -1:
            stop = len(items) - 1
        return items[start : stop + 1]

    def sadd(self, key: str, member: str) -> int:
        """Returns 1 if member was new, 0 if it already existed."""
        if key not in self._sets:
            self._sets[key] = set()
        if member in self._sets[key]:
            return 0
        self._sets[key].add(member)
        return 1

    def sismember(self, key: str, member: str) -> bool:
        return member in self._sets.get(key, set())


class EventTrackSystem:

    def __init__(self):
        # ── PostgreSQL simulation ──────────────────────────────────────
        self._pg_organizers: dict[str, dict] = {
            "org_001":   {"name": "TechConf Inc", "credits": 100},
            "org_empty": {"name": "Broke Org",    "credits": 0},
        }
        self._pg_registrations: dict[str, dict] = {}

        # Analytics log: fixed envelope + flexible JSONB payload
        self._analytics_log: list[dict] = []

        # ── Redis simulation ───────────────────────────────────────────
        self._redis = SimulatedRedis()

    # ── Public API ────────────────────────────────────────────────────

    def register_attendee(
        self,
        session_id: str,
        user_id: str,
        organizer_id: str,
        ticket_type: str,
        price_paid: float,
    ) -> dict:
        """
        Atomically deduct 1 credit AND save the registration.

        PostgreSQL equivalent (one transaction):
            BEGIN;
            UPDATE organizers
               SET credits = credits - 1
             WHERE id = %(organizer_id)s AND credits > 0;
            -- rowcount == 0 → ROLLBACK, raise InsufficientCredits
            INSERT INTO registrations (...) VALUES (...);
            COMMIT;
        """
        org = self._pg_organizers.get(organizer_id)
        if org is None:
            raise ValueError(f"Organizer '{organizer_id}' not found")
        if org["credits"] <= 0:
            raise ValueError(
                f"Organizer '{organizer_id}' has insufficient credits "
                f"(balance: {org['credits']})"
            )

        # Both of these succeed or neither does (simulated atomicity).
        org["credits"] -= 1
        registration = {
            "_id":          str(uuid.uuid4()),
            "session_id":   session_id,
            "user_id":      user_id,
            "organizer_id": organizer_id,
            "ticket_type":  ticket_type,
            "price_paid":   price_paid,
            "registered_at": time.time(),
        }
        self._pg_registrations[f"{session_id}:{user_id}"] = registration
        return registration

    def log_analytics_event(
        self,
        event_type: str,
        user_id: str,
        payload: dict,
    ) -> None:
        """
        Append any analytics event to the log (no migration for new types).

        PostgreSQL equivalent:
            INSERT INTO analytics_events (event_id, event_type, user_id, payload)
            VALUES (%(event_id)s, %(event_type)s, %(user_id)s, %(payload)s::jsonb)
            ON CONFLICT (event_id) DO NOTHING;   -- idempotency

        Redis: SET-based deduplication via SADD (returns 0 on duplicate).
        """
        # Use the caller's event_id if provided; otherwise generate a new one.
        # Callers that need idempotency MUST provide a stable event_id.
        event_id = payload.get("event_id") or str(uuid.uuid4())

        # Idempotency check: SADD returns 0 if event_id already processed.
        if self._redis.sadd("processed_events", event_id) == 0:
            return  # Duplicate — discard silently

        # Persist to analytics log (JSONB envelope + flexible payload)
        self._analytics_log.append({
            "event_id":   event_id,
            "event_type": event_type,
            "user_id":    user_id,
            "payload":    payload,
            "logged_at":  time.time(),
        })

        # ── Real-time leaderboard update (Constraint B) ──
        if event_type == "session_view" and "speaker_id" in payload:
            speaker_id = payload["speaker_id"]
            event_conf = payload.get("event_id_conf", "event_001")  # default for tests
            self._redis.zincrby(f"leaderboard:{event_conf}", 1.0, speaker_id)
            # ↑ ZINCRBY, not ZADD — adds 1 to current score

    def get_speaker_leaderboard(
        self, event_id: str, top_n: int = 10
    ) -> list[dict]:
        """
        O(log N) Redis Sorted Set read — never scans the analytics log.
        PostgreSQL equivalent would be:
            SELECT speaker_id, COUNT(*) AS score
            FROM analytics_events
            WHERE event_type = 'session_view'
            GROUP BY speaker_id
            ORDER BY score DESC
            LIMIT %(top_n)s;
        — which grows in cost as the analytics table grows.
        """
        key = f"leaderboard:{event_id}"
        top_items = self._redis.zrange(key, 0, top_n - 1, rev=True)
        return [
            {"speaker_id": member, "score": int(score)}
            for member, score in top_items
        ]

    def get_credit_balance(self, organizer_id: str) -> int:
        return self._pg_organizers.get(organizer_id, {}).get("credits", 0)
```

**Key Design Decisions in 3c:**

- **`sadd` for idempotency:** Redis `SADD` is atomic and returns `0` if the member already exists. Using a Redis Set to track processed `event_id` values is O(1) per check — no table scan, no lock. This mirrors how production event processors (Celery, Kafka consumers) deduplicate messages using a compact in-memory or TTL-backed store.
- **`zincrby` not `zadd`:** Each `session_view` event increments the speaker's score by 1. `ZINCRBY` is the semantically correct command (accumulate), whereas `ZADD` would replace the score (as Exercise 1.3 demonstrated).
- **Leaderboard as derived cache:** If Redis restarts, the leaderboard resets to zero. The raw events are still in the analytics log (PostgreSQL). Rebuilding the leaderboard is a one-time replay query. Redis is never the source of truth for analytics data — only for the pre-computed top-N view.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

**The Prompt:** A common developer misconception is that MongoDB is the right choice for any data that "looks like JSON." Describe a realistic scenario where a developer would *reasonably* reach for MongoDB, but where the choice is demonstrably worse than PostgreSQL. Write a small code snippet showing the failure mode, and explain the query performance difference at scale.

**Constraint:** The scenario must be one where the data genuinely has some flexible, JSON-like qualities — not an obviously relational use case. A junior developer should be able to see why they'd initially think MongoDB is appropriate.

**Questions:**

1. Describe the scenario in 2–3 sentences. Why might a developer initially choose MongoDB?

2. Write the PostgreSQL schema and the query that solves the core business question **in a single SQL statement**.

3. Write the same query using the document-store model (Python dicts as collections). Show that it requires **multiple application-level lookups**.

4. Compare query performance at scale: 500,000 employees, 5,000 departments, 200 divisions. Why does the document approach become significantly slower?

5. Name one additional PostgreSQL protection the document version loses, beyond JOIN performance.

---

## Level 4 Solution

**1. Scenario:**

An HR analytics platform tracks employees, departments, and divisions. The developer initially chose MongoDB because employee profiles have flexible attributes — contractors have `billing_rate`, full-timers have `equity_vest`, interns have `university` — and these vary wildly. So "each employee document has different fields" is a real, non-trivial argument for a document store. The core business query, however, is: *"Which employees are in a department whose division is the same as their manager's division?"*

**2. PostgreSQL Schema and Single Query:**

```sql
-- Schema
CREATE TABLE divisions   (id SERIAL PRIMARY KEY, name TEXT);
CREATE TABLE departments (id SERIAL PRIMARY KEY, name TEXT,
                          division_id INT REFERENCES divisions(id));
CREATE TABLE employees   (
    id            SERIAL PRIMARY KEY,
    name          TEXT,
    department_id INT REFERENCES departments(id),
    manager_id    INT REFERENCES employees(id),  -- self-referential FK
    profile       JSONB                          -- flexible attributes
);

-- The business query: one statement, uses indexes
SELECT
    e.name    AS employee,
    m.name    AS manager,
    dv.name   AS shared_division
FROM  employees  e
JOIN  employees  m  ON e.manager_id    = m.id          -- self-join
JOIN  departments d1 ON e.department_id = d1.id          -- employee's dept
JOIN  departments d2 ON m.department_id = d2.id          -- manager's dept
JOIN  divisions   dv ON d1.division_id  = dv.id
WHERE d1.division_id = d2.division_id                    -- same division
ORDER BY e.name;
```

PostgreSQL executes this with a plan using indexes on `manager_id` and `division_id`. Cost scales as O(E log D) where E = employees, D = departments.

**3. Document-Store Version:**

```python
# Three "collections" of dicts (MongoDB-style)
employees_col = {
    "e_001": {"name": "Alice",  "dept_id": "d_eng",  "manager_id": "e_005",
              "profile": {"type": "full_time", "equity_vest": "4yr"}},
    "e_002": {"name": "Bob",    "dept_id": "d_sales", "manager_id": "e_006",
              "profile": {"type": "contractor", "billing_rate": 150}},
}
departments_col = {
    "d_eng":   {"name": "Engineering", "division_id": "div_tech"},
    "d_sales": {"name": "Sales",       "division_id": "div_revenue"},
}

def find_employees_sharing_division_with_manager(employees, departments):
    results = []
    for emp_id, emp in employees.items():
        if not emp.get("manager_id"):
            continue

        # Lookup 1: employee's department
        emp_dept = departments.get(emp["dept_id"])

        # Lookup 2: manager record
        manager = employees.get(emp["manager_id"])
        if not manager:
            continue  # Dangling reference — no error raised!

        # Lookup 3: manager's department
        mgr_dept = departments.get(manager["dept_id"])

        if emp_dept and mgr_dept and emp_dept["division_id"] == mgr_dept["division_id"]:
            results.append({"employee": emp["name"], "manager": manager["name"]})

    return results
```

Every employee requires **3 separate lookups** — against employees (for the manager), departments (for the employee), and departments again (for the manager). With real MongoDB, each `find_one` is a network round trip.

**4. Performance at Scale:**

| | PostgreSQL | MongoDB (application-level) |
|---|---|---|
| Query pattern | 1 query, indexes on `division_id` and `manager_id` | 3N lookups (N = 500,000 employees) |
| Network round trips | 1 | Up to 1,500,000 |
| Scales as | O(E log D) | O(N) with massive I/O |
| Index usage | Planner uses all three FK indexes simultaneously | Each individual `find_one` uses its own index, called N times |

MongoDB's `$lookup` aggregation pipeline can avoid the round trips, but it is significantly more complex to write than the SQL JOIN, still slower than a native relational JOIN with proper indexes, and not type-safe.

**5. Additional Lost Protection:**

**Referential integrity.** In PostgreSQL, `manager_id INT REFERENCES employees(id)` prevents a manager from being deleted while employees still reference them. The document version silently produces dangling `manager_id` values (the `if not manager: continue` guard is evidence of this). PostgreSQL cannot produce this state; the constraint fires at write time. The document model produces it silently and requires the application to defend against it at read time.

---

# LEVEL 5: THE TEACH BACK

**Scenario:** You are doing code review. A junior developer opens a pull request:

> **Title:** "Migrate task manager to MongoDB — better performance, no migrations"
>
> **Description:** "MongoDB is faster than SQL because it doesn't have a fixed schema. Our task list endpoint is slow. Also, Alembic migrations keep breaking and I waste hours on them every sprint. If we use MongoDB, we never need migrations again. The REST API won't change."

They've started the migration:

```python
# ── BEFORE (PostgreSQL + SQLAlchemy) ──────────────────────────────────
class Task(Base):
    __tablename__ = "tasks"
    id:          Mapped[int]      = mapped_column(primary_key=True)
    title:       Mapped[str]      = mapped_column(String(255), nullable=False)
    assignee_id: Mapped[int|None] = mapped_column(ForeignKey("users.id"))
    category_id: Mapped[int|None] = mapped_column(ForeignKey("categories.id"))
    status:      Mapped[str]      = mapped_column(default="open")
    created_at:  Mapped[datetime] = mapped_column(default=datetime.utcnow)

# ── AFTER (proposed MongoDB document) ─────────────────────────────────
task_document = {
    "title":       "...",
    "assignee_id": "...",   # now just a plain string
    "category_id": "...",   # now just a plain string
    "status":      "open",
}
```

Write your code review comment. Address each of the developer's claims with **specific technical arguments**. Cover:

1. **The "MongoDB is faster" claim** — Is this categorically true? What is the actual cause of the slow endpoint, and what is the targeted fix?
2. **The "no migrations" claim** — Is it accurate? What does moving to MongoDB replace migrations with, and is that better or worse?
3. **The foreign key regression** — What specific protection does the `ForeignKey` in the SQLAlchemy model provide that the proposed document schema loses? Name the exact mechanism and the exact failure mode it prevents.
4. **The "API won't change" claim** — Identify one query the existing API performs that becomes materially harder in MongoDB.
5. **The judgment call** — Is there any part of the task manager that would legitimately benefit from MongoDB or Redis? Give a specific, defensible answer.

---

## Level 5 Solution

**Code Review Comment:**

Thanks for thinking about performance — it's the right instinct. But let me work through each claim before we merge this, because I think the diagnosis and the cure are mismatched here.

---

**1. "MongoDB is faster than SQL."**

This is not categorically true. Database performance is determined almost entirely by access patterns and indexes, not by the database type.

The slow endpoint is almost certainly caused by one of three things you can check in ten minutes:

- A missing index on `(status, assignee_id)` — add it with a two-line Alembic migration and run `EXPLAIN ANALYZE` before and after.
- An N+1 query — loading 50 tasks and then issuing a separate `SELECT` per task to fetch the assignee name. Fix with `selectinload(Task.assignee)` in SQLAlchemy.
- Missing pagination — returning 5,000 tasks when the UI shows 20. Fix with `.limit(20).offset(page * 20)`.

MongoDB is not faster for structured, relational queries. A PostgreSQL B-tree index on `(status, assignee_id)` serves a filtered task list in under 5ms. A MongoDB query over the same data is comparable — and you'd lose native JOIN support in the process.

**Concrete next step:** Add `EXPLAIN ANALYZE` to the slow query. Find the `Seq Scan`. Add the missing index. This takes 30 minutes.

---

**2. "No migrations with MongoDB."**

Partially accurate — you won't write `ALTER TABLE`. But this is what you'll do instead:

- Write Pydantic models for every document shape — the schema still exists, it just lives in Python now.
- Write data-migration scripts when you rename a field — there's no `alembic downgrade`, no automatic rollback, and no tooling to apply the change to existing documents.
- Debug "why is this field present in new documents but missing from documents created last year" — this is called schema drift and Exercise 1.1 from this week shows exactly how it happens silently.

Alembic migrations feel painful because they make schema changes **explicit, versioned, and reversible**. That friction is a feature. MongoDB pushes the pain to the debugging phase — later, harder, and with real users affected.

---

**3. The Foreign Key Regression.**

Look at what was just removed:

```python
# BEFORE — this line is doing real work:
assignee_id: Mapped[int|None] = mapped_column(ForeignKey("users.id"))

# AFTER — this is just a string field with no enforcement:
"assignee_id": "...",
```

`ForeignKey("users.id")` instructs PostgreSQL to enforce referential integrity. When you delete a user, PostgreSQL can be configured to either:
- Prevent the delete if tasks still reference that user (`RESTRICT`), or
- Automatically null out `assignee_id` on those tasks (`ON DELETE SET NULL`), or
- Cascade the delete to the tasks (`ON DELETE CASCADE`).

In the MongoDB version, `assignee_id` is just a string. If you delete a user, every task that referenced them now holds a dangling ID pointing at nothing. The next time `get_open_tasks_with_assignees` runs, it calls `users.find_one(task["assignee_id"])`, gets `None`, and crashes with `TypeError: 'NoneType' object is not subscriptable`. Exercise 2.2 this week demonstrates this exact failure mode with runnable code.

PostgreSQL **cannot** produce this state because the constraint fires at delete time, regardless of which connection or tool performs the delete. The document model produces it silently.

---

**4. "The API won't change."**

Consider the existing `GET /tasks?category=backend&assigned_to=alice@corp.com` endpoint. In SQLAlchemy today:

```python
query = (
    select(Task)
    .join(User,     Task.assignee_id == User.id)
    .join(Category, Task.category_id == Category.id)
    .where(Category.name == "backend")
    .where(User.email == "alice@corp.com")
)
```

One database round trip. One query plan. PostgreSQL uses indexes on both FK columns.

In MongoDB — no native JOIN — you'd need to:
1. Query `users` for Alice's `_id` by email.
2. Query `categories` for "backend"'s `_id`.
3. Query `tasks` with those two IDs as filters.

Three round trips per request, or one complex `$lookup` aggregation pipeline. The task manager is fundamentally relational: users have tasks, tasks have categories, tasks have tags, tasks have comments. The lecture's "When Relational is Still King" section describes this domain precisely.

---

**5. The Judgment Call.**

For the current task manager, MongoDB is not the right answer — but Redis is, and we'll add it in Week 10. Here's where each tool actually belongs:

| Feature | Right Tool | Why |
|---|---|---|
| Tasks, users, categories | PostgreSQL ✅ (already correct) | Clear FK relationships, ACID needed |
| Task list caching | Redis (Week 10) | Sub-ms repeated reads, TTL expiration |
| Login rate limiting | Redis (Week 10) | Atomic `INCR`, auto-expire after window |
| WebSocket broadcast | Redis pub/sub (Week 12) | Multi-server real-time messaging |
| Task `metadata` (flexible per task type) | PostgreSQL JSONB column | Already in PostgreSQL, no new database needed |

The right answer to a slow endpoint is **targeted optimization**, not a database replacement. The right time to add Redis is when we've measured a specific latency problem that PostgreSQL genuinely can't solve. The right time to add MongoDB is if we were building a product catalog with wildly varying attribute schemas — not a task manager with stable, well-defined entities.