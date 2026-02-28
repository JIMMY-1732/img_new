# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "A database is basically a Python dictionary saved to disk. I can get the same guarantees by writing my dict to a JSON file and adding some validation code. Tables are just dicts-of-dicts, relationships are just matching IDs, and if I validate with Pydantic at the API boundary, my data is safe."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | Pydantic validation at the API layer guarantees data is always valid in storage | Internal code, migrations, bulk imports, and direct writes bypass Pydantic entirely; the storage layer has no guards |
| 2 | Deleting a referenced record will raise an error or automatically clean up related data | Python dicts have zero awareness of cross-dict references; delete silently succeeds, orphaning anything that pointed to it |
| 3 | Concurrent async requests each see a consistent, up-to-date view of shared state | Each coroutine captures a local snapshot; between `await` points, other coroutines may have already mutated the shared state |
| 4 | One big table (dict) is simpler and produces the same results as multiple related tables | Denormalized storage causes update, insert, and delete anomalies that compound silently over time |
| 5 | Data types don't matter much as long as the values "look right" as strings | Lexicographic comparison differs from numeric comparison: `"9.99" > "20.00"` is `True`; `"100" < "25"` is `True` |
| 6 | You can enforce uniqueness and referential integrity in application code and get the same safety as database constraints | Application-level checks suffer from Time-of-Check-to-Time-of-Use (TOCTOU) race conditions; a gap always exists between checking and writing |
| 7 | ACID is a theoretical textbook concept, not a practical engineering concern | Every multi-step operation — transfers, orders, bookings — will silently corrupt data without atomicity and isolation |

**The Prediction Gap:** When shown concurrent async operations on a shared dict "database," students default to a sequential mental model and predict a correct final state. The actual behavior — all coroutines reading the same stale snapshot before any write executes — produces a state that is numerically impossible (e.g., more items "sold" than existed in stock), yet triggers zero runtime errors.

---

## The Main Purpose (Essence)

**Core Insight:** A relational database is not "a dict that doesn't forget." It is a system that makes four mathematically enforced guarantees (ACID) about your data that no amount of application code can fully replicate. The purpose of this lecture is to make students *feel the pain* of operating without those guarantees, so that every database concept — tables, keys, types, constraints, normalization — is understood as a **solution to a specific, demonstrated failure mode.**

**The Problems Being Solved:**

| Concept | Without It (Python dict) | With It (Relational Database) |
|---------|--------------------------|-------------------------------|
| Persistence (Durability) | Server restart = total data loss | Data survives crashes, restarts, power outages |
| Typed Columns | Any value in any field; `"banana"` in an age field | Engine rejects type mismatches at write time |
| Constraints (NOT NULL, UNIQUE, CHECK) | Enforced only if your code remembers to check | Enforced by the engine; no code path can bypass |
| Foreign Keys | Orphaned references, phantom associations, silent corruption | References guaranteed valid; invalid writes rejected |
| Normalization | Update anomalies, data duplication, conflicting facts | Each fact stored once; consistency by structure |
| ACID Transactions | Race conditions, partial failures, lost updates | All-or-nothing operations; isolated concurrent access |

**The Key Distinction:**
- **Pydantic** validates data **at the API boundary** (input from users).
- **Database constraints** validate data **at the storage boundary** (what gets persisted).
- **Neither alone is sufficient.** Pydantic can be bypassed by internal code. Database constraints cannot be bypassed by anything. Both layers together form defense in depth.

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "Async requests run one-at-a-time" | Only 2 of 3 orders confirmed (stock is sufficient for only 2) | All 3 confirmed; stock shows an impossible value | All coroutines read the same snapshot before any write executes |
| "Delete fails if other data references it" | Error raised when deleting a referenced category | Delete silently succeeds; tasks now point at nothing — or worse, at something new | Python dicts have no cross-reference awareness |
| "Updating a user's email updates it everywhere" | All records show the new email after update | Some records show old email, some show new | Denormalized storage stores the same fact in multiple places; a partial update creates split-brain data |
| "Sorting string-numbers works like sorting real numbers" | Products sorted correctly by price | `$100.00` sorted before `$8.99`; `$9.99` excluded from "under $20" filter | Lexicographic comparison: `"1" < "8" < "9"`, regardless of the numeric value the string represents |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Phantom Inventory

```python
import asyncio

inventory = {"widget_a": 10}
orders_processed: list[tuple[str, str]] = []


async def process_order(order_id: str, item: str, quantity: int) -> None:
    # Step 1: Read current stock
    current_stock = inventory[item]

    # Step 2: Validate availability
    if current_stock < quantity:
        orders_processed.append((order_id, "REJECTED"))
        return

    # Step 3: Simulate payment processing (I/O)
    await asyncio.sleep(0.01)

    # Step 4: Deduct stock and confirm
    inventory[item] = current_stock - quantity
    orders_processed.append((order_id, "CONFIRMED"))


async def main() -> None:
    # 3 orders, each wants 4 widgets (only 10 in stock)
    await asyncio.gather(
        process_order("ORD-1", "widget_a", 4),
        process_order("ORD-2", "widget_a", 4),
        process_order("ORD-3", "widget_a", 4),
    )

    confirmed = [oid for oid, s in orders_processed if s == "CONFIRMED"]
    rejected = [oid for oid, s in orders_processed if s == "REJECTED"]

    print(f"Final stock: {inventory['widget_a']}")
    print(f"Confirmed: {confirmed}")
    print(f"Rejected:  {rejected}")
    print(f"Items sold on paper: {len(confirmed) * 4}")
    print(f"Items actually deducted from stock: {10 - inventory['widget_a']}")


asyncio.run(main())
```

**Questions:**

1. **Predict the output.** Specifically: how many orders are confirmed? How many rejected? What is the final stock? How many items were actually deducted?

2. **Explain the discrepancy.** A developer says: "I check `current_stock < quantity` before deducting, so it's impossible to sell more than we have." Why is the developer wrong?

3. **Mental trace:** Walk through what happens at each `await` boundary. At the moment each coroutine resumes after `asyncio.sleep`, what value does its local variable `current_stock` hold? Is this the same as `inventory["widget_a"]` at that moment?

4. **Connect to ACID.** Which ACID property, if enforced, would prevent this corruption? What would the correct behavior look like?

---

## Exercise 1.2: The Identity Thief

```python
categories: dict[int, dict] = {
    1: {"id": 1, "name": "Backend",  "color": "#3B82F6"},
    2: {"id": 2, "name": "Frontend", "color": "#10B981"},
    3: {"id": 3, "name": "DevOps",   "color": "#F59E0B"},
}

tasks: list[dict] = [
    {"id": 101, "title": "Build API",      "category_id": 1},
    {"id": 102, "title": "Write tests",    "category_id": 1},
    {"id": 103, "title": "Fix navbar",     "category_id": 2},
    {"id": 104, "title": "Setup CI/CD",    "category_id": 3},
    {"id": 105, "title": "Deploy service",  "category_id": 3},
]


def display_tasks() -> None:
    for task in tasks:
        cat = categories.get(task["category_id"])
        cat_name = cat["name"] if cat else "[DELETED]"
        print(f"  {task['title']} [{cat_name}]")


# Step 1: Manager says "Delete the DevOps category"
print("=== Step 1: Delete DevOps ===")
del categories[3]
display_tasks()

# Step 2: Team creates a new "Security" category
print("\n=== Step 2: Add 'Security' category ===")
categories[3] = {"id": 3, "name": "Security", "color": "#EF4444"}
display_tasks()

# Step 3: Audit report
print("\n=== Step 3: Audit ===")
for task in tasks:
    cat = categories.get(task["category_id"])
    if cat:
        print(f"  Task {task['id']} belongs to: {cat['name']}")
```

**Questions:**

1. **Predict the output for all three steps.** What category do tasks 104 and 105 show in Step 1? In Step 2? In Step 3?

2. **Explain the phantom association.** Nobody ran any code to reassign tasks 104 or 105. Why do they now claim to belong to "Security"?

3. **Identify three distinct failures** that a relational database's foreign key constraint would have prevented in this sequence.

4. **Compare behaviors.** A database foreign key with `ON DELETE RESTRICT` would behave differently at Step 1. A foreign key with `ON DELETE CASCADE` would also behave differently but in another way. Describe both behaviors.

---

## Exercise 1.3: The Email Nobody Got

```python
task_records: list[dict] = [
    {"task_id": 1, "title": "Build API",   "assignee": "Alice", "assignee_email": "alice@old.com"},
    {"task_id": 2, "title": "Write tests", "assignee": "Alice", "assignee_email": "alice@old.com"},
    {"task_id": 3, "title": "Fix navbar",  "assignee": "Bob",   "assignee_email": "bob@co.com"},
    {"task_id": 4, "title": "Setup CI",    "assignee": "Alice", "assignee_email": "alice@old.com"},
]


def update_user_email(name: str, new_email: str) -> int:
    """Update a user's email. Returns number of records updated."""
    updated = 0
    for record in task_records:
        if record["assignee"] == name:
            record["assignee_email"] = new_email
            updated += 1
            break  # Found the user, done!
    return updated


# Alice got a new corporate email
result = update_user_email("Alice", "alice@new.com")
print(f"Records updated: {result}")

# Check state
alice_emails = set(
    r["assignee_email"] for r in task_records if r["assignee"] == "Alice"
)
print(f"Alice's emails in system: {alice_emails}")
print(f"Number of distinct emails: {len(alice_emails)}")

# Send task notifications
print("\nSending notifications:")
for record in task_records:
    if record["assignee"] == "Alice":
        print(f"  '{record['title']}' -> {record['assignee_email']}")
```

**Questions:**

1. **Predict the output.** How many records were updated? How many distinct emails does Alice have? Which email does each notification go to?

2. **Identify the surface bug AND the structural flaw.** The developer used `break` thinking "I found the user, done." That's the bug. But even if you remove `break` and fix the immediate behavior, why is this data structure *still* fundamentally broken?

3. **Demonstrate with a scenario.** Suppose the `break` is removed and the update works correctly for all records. Now describe a realistic sequence of events (involving two developers or two API endpoints) that re-introduces inconsistency in this denormalized structure.

4. **Redesign.** Sketch (in pseudocode or dict structures) the normalized version of this data. How many "tables" (dicts) do you need? Where does Alice's email live, and how many times?

---

## Exercise 1.4: The Price Is Wrong

```python
products: list[dict] = [
    {"name": "Widget A",  "price": "9.99",   "stock": "100"},
    {"name": "Widget B",  "price": "10.50",  "stock": "25"},
    {"name": "Widget C",  "price": "9.50",   "stock": "8"},
    {"name": "Widget D",  "price": "100.00", "stock": "50"},
    {"name": "Widget E",  "price": "8.99",   "stock": "1000"},
]

# Sort by price (cheapest first)
by_price = sorted(products, key=lambda p: p["price"])
print("Sorted by price (cheapest first):")
for p in by_price:
    print(f"  {p['name']}: ${p['price']}")

# Sort by stock (most in stock first)
by_stock = sorted(products, key=lambda p: p["stock"], reverse=True)
print("\nSorted by stock (most first):")
for p in by_stock:
    print(f"  {p['name']}: {p['stock']} units")

# Filter: products under $20
cheap = [p for p in products if p["price"] < "20.00"]
print("\nProducts under $20.00:")
for p in cheap:
    print(f"  {p['name']}: ${p['price']}")

# Reveal
print(f"\nString: '9.99' < '20.00' = {'9.99' < '20.00'}")
print(f"Number:  9.99  <  20.00  = {9.99 < 20.00}")
print(f"String: '100.00' < '20.00' = {'100.00' < '20.00'}")
print(f"Number:  100.00  <  20.00  = {100.00 < 20.00}")
```

**Questions:**

1. **Predict the output.** Write the exact sorted order for both sorts, and the exact list of "products under $20." Most students get at least one of these wrong.

2. **Explain the sorting.** Why does `$100.00` appear *before* `$8.99` in the price sort? Why does `8 units` appear as "more stock" than `1000 units`?

3. **Explain the filter.** Why is `$100.00` "under $20" but `$9.99` is *not* "under $20"? Walk through the character-by-character comparison.

4. **Connect to databases.** The lecture says "use `NUMERIC(p,s)` for money" and "use `INTEGER` for counts." How would using proper PostgreSQL column types prevent every single bug demonstrated above?

---

# Level 1 Solutions

## Solution 1.1: The Phantom Inventory

**1. Output:**
```
Final stock: 6
Confirmed: ['ORD-1', 'ORD-2', 'ORD-3']
Rejected:  []
Items sold on paper: 12
Items actually deducted from stock: 4
```

**2. Explanation:**

The developer's reasoning is correct *for sequential execution*: if orders run one-at-a-time, only 2 can be confirmed (4+4=8 ≤ 10; third sees 2 < 4, rejected). But `asyncio.gather` starts all three coroutines concurrently. The `await asyncio.sleep` creates a yield point where the event loop can switch between coroutines.

**3. Mental Trace:**
```
T0: Event loop starts all 3 coroutines.

ORD-1 executes:
  current_stock = inventory["widget_a"]  → local variable = 10
  10 < 4? No.
  await asyncio.sleep(0.01)              → YIELDS to event loop.

ORD-2 executes:
  current_stock = inventory["widget_a"]  → local variable = 10 (UNCHANGED)
  10 < 4? No.
  await asyncio.sleep(0.01)              → YIELDS.

ORD-3 executes:
  current_stock = inventory["widget_a"]  → local variable = 10 (STILL UNCHANGED)
  10 < 4? No.
  await asyncio.sleep(0.01)              → YIELDS.

~0.01s later, all three wake up, run to completion without yielding:

ORD-1 resumes:
  inventory["widget_a"] = 10 - 4 = 6    → writes 6
  Appends ("ORD-1", "CONFIRMED")

ORD-2 resumes:
  inventory["widget_a"] = 10 - 4 = 6    → writes 6 (its local current_stock is still 10!)
  Appends ("ORD-2", "CONFIRMED")

ORD-3 resumes:
  inventory["widget_a"] = 10 - 4 = 6    → writes 6 (same stale snapshot)
  Appends ("ORD-3", "CONFIRMED")
```

After all three: inventory = 6. All confirmed. 12 items "sold," 4 actually deducted. 8 phantom items materialized from nowhere.

The critical insight: `current_stock` is a **local variable** captured before the `await`. After resuming, it is stale — it does not reflect writes made by other coroutines during the suspension. Each coroutine computes `10 - 4 = 6` independently and overwrites the same key.

**4. ACID Connection:**

**Isolation** prevents this. With proper isolation, each concurrent operation would behave as if it were the only one running. The database would serialize the read-check-write sequence so that ORD-2 sees the updated stock (6) after ORD-1's write, checks `6 >= 4` (passes), and writes 2. ORD-3 then sees 2, checks `2 < 4`, and is rejected.

Expected result under Isolation: 2 confirmed, 1 rejected, stock = 2.

---

## Solution 1.2: The Identity Thief

**1. Output:**

```
=== Step 1: Delete DevOps ===
  Build API [Backend]
  Write tests [Backend]
  Fix navbar [Frontend]
  Setup CI/CD [DELETED]
  Deploy service [DELETED]

=== Step 2: Add 'Security' category ===
  Build API [Backend]
  Write tests [Backend]
  Fix navbar [Frontend]
  Setup CI/CD [Security]
  Deploy service [Security]

=== Step 3: Audit ===
  Task 101 belongs to: Backend
  Task 102 belongs to: Backend
  Task 103 belongs to: Frontend
  Task 104 belongs to: Security
  Task 105 belongs to: Security
```

**2. Phantom Association:**

Tasks 104 and 105 store `category_id: 3` — just a bare integer. They have no knowledge of what that integer *means.* When category 3 was "DevOps," the lookup resolved to DevOps. When category 3 is deleted, the lookup returns `None`. When a *new* category is assigned ID 3, the *same unchanged* integer now resolves to "Security."

Nobody touched the tasks. The *meaning* of their reference changed from underneath them. This is **phantom association through ID reuse**: the integer 3 is the same, but the entity it identifies is completely different.

**3. Three failures a foreign key would prevent:**

- **Step 1 — `ON DELETE RESTRICT`:** The `DELETE FROM categories WHERE id=3` would be **rejected** because rows in `tasks` still reference `category_id=3`. You are forced to handle the tasks first (reassign or delete them).
- **Step 1 — `ON DELETE CASCADE`:** The delete would **succeed** but automatically delete tasks 104 and 105 as well. No orphans left behind.
- **Step 2 — ID reuse prevention:** With `SERIAL` or `UUID` primary keys, ID 3 would never be reused. The new category would get ID 4 (serial) or a new UUID. Even if orphans existed, they could never accidentally point to a *different* entity.

**4. Behavior Comparison:**

`ON DELETE RESTRICT`: Step 1 fails with an error. Categories dict remains unchanged. All 5 tasks keep valid references. The developer is forced to decide what to do with the DevOps tasks before the category can be removed.

`ON DELETE CASCADE`: Step 1 succeeds, **and** tasks 104 and 105 are automatically deleted. After Step 1, only 3 tasks remain, all with valid references. Step 2 would work fine because nothing references the old ID.

---

## Solution 1.3: The Email Nobody Got

**1. Output:**
```
Records updated: 1
Alice's emails in system: {'alice@old.com', 'alice@new.com'}
Number of distinct emails: 2

Sending notifications:
  'Build API' -> alice@new.com
  'Write tests' -> alice@old.com
  'Setup CI' -> alice@old.com
```

Only 1 record updated. Alice now has **two different emails** in the system. Two out of three notifications go to the wrong address.

**2. Surface Bug vs. Structural Flaw:**

**Surface bug:** The `break` statement causes the loop to stop after the first match. The developer thought "I found Alice" when they should have thought "I found *one row containing Alice*." Removing `break` fixes the immediate symptom — all three records get updated.

**Structural flaw:** Alice's email is stored in **three separate places.** Even without the `break`, this structure is fundamentally fragile. The normalization violation (Alice's email as a repeated denormalized field across task records) means any operation that writes Alice's email must find and update *every copy*, perfectly, every time. This is the **Update Anomaly** from the lecture.

**3. Realistic failure scenario (even with `break` removed):**

Developer A writes an endpoint: `PUT /users/{id}/email` that calls `update_user_email()` (now fixed, no `break`). Developer B writes a separate endpoint: `POST /tasks/import` that creates tasks from a CSV file. The CSV contains `{"title": "New task", "assignee": "Alice", "assignee_email": "alice@old.com"}` — stale data from an old export. After import, Alice has a mix of old and new emails again. No single developer did anything wrong. The *structure* made inconsistency inevitable.

**4. Normalized Redesign:**

```python
# Table 1: users — Alice's email stored ONCE
users = {
    1: {"id": 1, "name": "Alice", "email": "alice@new.com"},
    2: {"id": 2, "name": "Bob",   "email": "bob@co.com"},
}

# Table 2: tasks — references user by ID, stores NO user details
tasks = {
    1: {"id": 1, "title": "Build API",   "user_id": 1},
    2: {"id": 2, "title": "Write tests", "user_id": 1},
    3: {"id": 3, "title": "Fix navbar",  "user_id": 2},
    4: {"id": 4, "title": "Setup CI",    "user_id": 1},
}
```

Alice's email lives in **one place.** Updating it requires changing **one record.** Every task references Alice by `user_id=1`; the email is resolved at query time by looking up `users[1]`. No duplication. No anomaly. This is Third Normal Form.

---

## Solution 1.4: The Price Is Wrong

**1. Output:**
```
Sorted by price (cheapest first):
  Widget B: $10.50
  Widget D: $100.00
  Widget E: $8.99
  Widget C: $9.50
  Widget A: $9.99

Sorted by stock (most first):
  Widget C: 8 units
  Widget D: 50 units
  Widget B: 25 units
  Widget E: 1000 units
  Widget A: 100 units

Products under $20.00:
  Widget B: $10.50
  Widget D: $100.00

String: '9.99' < '20.00' = False
Number:  9.99  <  20.00  = True
String: '100.00' < '20.00' = True
Number:  100.00  <  20.00  = False
```

**2. Sorting Explanation:**

String sorting is **lexicographic** — character by character, left to right, using ASCII values. The string `"100.00"` starts with `"1"` (ASCII 49). The string `"8.99"` starts with `"8"` (ASCII 56). Since `49 < 56`, `"100.00" < "8.99"` regardless of what the rest of the string says. The sort never sees the full numeric value.

For stock: `"8"` starts with `"8"` (ASCII 56), which is greater than `"5"` (50), `"2"` (50), and `"1"` (49). So `"8"` sorts *last* ascending and *first* descending — meaning 8 units is "the most."

**3. Filter Walkthrough:**

`"100.00" < "20.00"` — compare character by character:
- Position 0: `"1"` (49) vs `"2"` (50) → `"1" < "2"` → **True**. Comparison stops.

Python concludes `"100.00" < "20.00"`. The `$100` product passes the "under $20" filter.

`"9.99" < "20.00"` — compare:
- Position 0: `"9"` (57) vs `"2"` (50) → `"9" > "2"` → **False**. Comparison stops.

Python concludes `"9.99"` is *not* less than `"20.00"`. The `$9.99` product is excluded from "under $20."

**4. Database Type Connection:**

If `price` were `NUMERIC(10,2)` (PostgreSQL), the database stores the *actual numeric value*, not a string representation. Comparisons use numeric math: `9.99 < 20.00` is `True`; `100.00 < 20.00` is `False`. Sorting produces correct numeric order. Filtering works correctly.

If `stock` were `INTEGER`, `8 < 1000` is `True` (numeric), and sorting by stock gives the correct order. Every bug in this exercise — wrong sort order, wrong filter results, inverted comparisons — stems from one root cause: treating numeric data as strings. Proper column types eliminate the entire class of bugs.

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Validation Window

**Context:** After learning about referential integrity, a developer adds a "user exists" check to the task creation endpoint. They're convinced this is equivalent to a foreign key constraint.

```python
import asyncio

# Our "database"
users: dict[int, str] = {1: "Alice", 2: "Bob", 3: "Charlie"}
tasks: dict[int, dict] = {}
next_task_id: int = 1


async def create_task(title: str, user_id: int) -> dict:
    """Create a task. Validates that the user exists first."""
    global next_task_id

    # Referential integrity check — just like a foreign key, right?
    if user_id not in users:
        raise ValueError(f"User {user_id} does not exist — integrity enforced!")

    # Simulate I/O: writing to storage
    await asyncio.sleep(0.01)

    task = {"id": next_task_id, "title": title, "user_id": user_id}
    tasks[next_task_id] = task
    next_task_id += 1
    return task


async def delete_user(user_id: int) -> str:
    """Delete a user immediately."""
    if user_id not in users:
        return f"User {user_id} not found"
    name = users.pop(user_id)
    return f"Deleted {name}"


async def main() -> None:
    task_result, delete_result = await asyncio.gather(
        create_task("Charlie's critical report", user_id=3),
        delete_user(3),
        return_exceptions=True,
    )

    print(f"Task creation result: {task_result}")
    print(f"User deletion result: {delete_result}")
    print(f"\nCurrent users: {users}")
    print(f"Current tasks: {tasks}")

    print("\nIntegrity audit:")
    for tid, task in tasks.items():
        owner = users.get(task["user_id"])
        status = f"owner='{owner}'" if owner else "ORPHANED — owner does not exist!"
        print(f"  Task {tid} ('{task['title']}'): {status}")


asyncio.run(main())
```

---

**Task 2a: Identify the Flaw**

The code has an explicit `if user_id not in users` check. The developer claims this enforces referential integrity.

1. Run the code. Does a task get created? Does the integrity check prevent the orphan?
2. What is the output of the integrity audit?

Do NOT fix the code yet. Just identify why the check fails.

---

**Task 2b: Explain the Mechanism**

This is a classic TOCTOU (Time-of-Check to Time-of-Use) vulnerability:

1. At what exact moment does `create_task` verify that user 3 exists?
2. At what exact moment does `create_task` actually write the task referencing user 3?
3. What happens between those two moments?
4. Why can't application-level code eliminate this gap?

---

**Task 2c: Instrument to Prove Diagnosis**

Add `print` statements inside both `create_task` and `delete_user` to show the exact timeline:

```python
# Inside create_task, after the check:
print(f"[create_task] User {user_id} verified at check time")
# After the await:
print(f"[create_task] Writing task now. User {user_id} still exists? {user_id in users}")

# Inside delete_user, before pop:
print(f"[delete_user] Deleting user {user_id} NOW")
```

Show the output that proves the check happened BEFORE the deletion and the write happened AFTER.

---

**Task 2d: Fix the Code**

Fix this using `asyncio.Lock` to simulate database-level transactional protection. Requirements:

1. The check and the write must be **atomic** (no yield between them).
2. `delete_user` must also check that no tasks currently reference the user before deleting (simulating `ON DELETE RESTRICT`).
3. After your fix, running the same concurrent scenario should result in either: (a) the task is created and the delete is blocked, OR (b) the user is deleted and the task creation fails. Never both succeeding.

---

**Task 2e: Write Verification Test**

```python
async def test_no_orphans_under_concurrency():
    """After any combination of concurrent creates and deletes,
    every task must reference an existing user."""
    # Reset state
    global users, tasks, next_task_id
    users = {1: "Alice", 2: "Bob", 3: "Charlie"}
    tasks = {}
    next_task_id = 1

    # Run concurrent create + delete
    await asyncio.gather(
        create_task("Test task", user_id=3),
        delete_user(3),
        return_exceptions=True,
    )

    # INVARIANT: every task must reference an existing user
    for tid, task in tasks.items():
        assert task["user_id"] in users, (
            f"ORPHAN DETECTED: Task {tid} references user {task['user_id']} "
            f"who does not exist!"
        )
    print("PASSED: No orphaned tasks.")
```

This test **FAILS** on the original code and **PASSES** on your fixed version.

---

## Exercise 2.2: The Counter That Drifted

**Context:** A developer added `task_count` to each category dict to speed up dashboard rendering. The feature works perfectly — until it doesn't.

```python
categories: dict[int, dict] = {
    1: {"id": 1, "name": "Backend",  "task_count": 0},
    2: {"id": 2, "name": "Frontend", "task_count": 0},
    3: {"id": 3, "name": "Testing",  "task_count": 0},
}

tasks: dict[int, dict] = {}
next_task_id: int = 1


def create_task(title: str, category_id: int) -> dict:
    """Create a task and update the category's cached count."""
    global next_task_id
    if category_id not in categories:
        raise ValueError(f"Category {category_id} not found")
    task = {"id": next_task_id, "title": title, "category_id": category_id}
    tasks[next_task_id] = task
    next_task_id += 1
    categories[category_id]["task_count"] += 1
    return task


def delete_task(task_id: int) -> bool:
    """Delete a task and update the category's cached count."""
    if task_id not in tasks:
        return False
    cat_id = tasks[task_id]["category_id"]
    del tasks[task_id]
    categories[cat_id]["task_count"] -= 1
    return True


def reassign_task(task_id: int, new_category_id: int) -> bool:
    """Move a task to a different category."""
    if task_id not in tasks or new_category_id not in categories:
        return False
    tasks[task_id]["category_id"] = new_category_id
    return True


def dashboard() -> None:
    """Display dashboard using cached counts (fast!) vs actual counts."""
    print("  Category Dashboard:")
    for cid, cat in categories.items():
        actual = sum(1 for t in tasks.values() if t["category_id"] == cid)
        match = "✓" if cat["task_count"] == actual else f"✗ DRIFT (actual: {actual})"
        print(f"    {cat['name']}: cached={cat['task_count']}, {match}")


# Phase 1: Standard CRUD (works correctly)
create_task("Build API", 1)
create_task("Write endpoints", 1)
create_task("Design DB schema", 1)
create_task("Fix navbar", 2)
create_task("Unit tests", 3)
create_task("Integration tests", 3)

print("After standard creates:")
dashboard()

# Phase 2: Reassign a task
print("\nReassigning 'Design DB schema' (id=3) from Backend to Testing...")
reassign_task(3, 3)

print("\nAfter reassignment:")
dashboard()

# Phase 3: Delete the reassigned task
print("\nDeleting 'Design DB schema' (id=3)...")
delete_task(3)

print("\nAfter deletion:")
dashboard()
```

---

**Task 2a: Identify the Flaw**

1. Run the code. Which phase causes the counts to drift?
2. Which function is missing the cache update?
3. After the deletion in Phase 3, do the counts drift further or self-correct? Why?

---

**Task 2b: Explain the Mechanism**

1. After `reassign_task(3, 3)`, what does Backend's `task_count` say? What is the actual count?
2. After `delete_task(3)`, which category's `task_count` gets decremented? Is that the right one?
3. Trace the compound error: the deletion uses `tasks[3]["category_id"]` to find which category to decrement. After the reassignment, this is `3` (Testing). So Testing's count goes from `2 → 1`, but the actual is `2`. Meanwhile Backend's count stays at `3`, but the actual is `2`. Explain why both counts are now wrong and in opposite directions.

---

**Task 2c: Instrument to Prove Diagnosis**

Add a call to `dashboard()` after EVERY operation (every create, the reassignment, and the delete). Show the exact point where drift begins and how it compounds.

---

**Task 2d: Fix the Code**

Provide **two** different fixes:

**Fix A (Bandaid):** Modify `reassign_task` to update both the old and new category counts. This fixes the immediate bug.

**Fix B (Normalization):** Remove `task_count` from categories entirely. Modify `dashboard()` to always compute the count dynamically. Explain why this is the superior fix by connecting to the lecture's normalization concepts.

---

**Task 2e: Write Verification Test**

```python
def test_counts_always_consistent():
    """Cached count must equal actual count after ANY sequence of operations."""
    # (reset state before test)
    create_task("Task A", 1)
    create_task("Task B", 1)
    create_task("Task C", 2)
    reassign_task(1, 2)      # Move Task A from Backend to Frontend
    reassign_task(2, 2)      # Move Task B from Backend to Frontend
    delete_task(3)            # Delete Task C

    for cid, cat in categories.items():
        actual = sum(1 for t in tasks.values() if t["category_id"] == cid)
        assert cat["task_count"] == actual, (
            f"DRIFT: {cat['name']} cached={cat['task_count']}, actual={actual}"
        )
    print("PASSED: All counts consistent.")
```

This test FAILS on original code and PASSES on Fix A. For Fix B, rewrite the test to check the return value of `dashboard()` instead (since `task_count` no longer exists as a cached field).

---

# Level 2 Solutions

## Solution 2.1: The Validation Window

**Task 2a:**

Output:
```
Task creation result: {'id': 1, 'title': "Charlie's critical report", 'user_id': 3}
User deletion result: Deleted Charlie

Current users: {1: 'Alice', 2: 'Bob'}
Current tasks: {1: {'id': 1, 'title': "Charlie's critical report", 'user_id': 3}}

Integrity audit:
  Task 1 ('Charlie's critical report'): ORPHANED — owner does not exist!
```

The check did NOT prevent the orphan. The task was created AND the user was deleted. Both operations succeeded.

**Task 2b:**

1. The check (`user_id not in users`) runs at the **start** of `create_task`, before the `await`.
2. The actual write (`tasks[next_task_id] = task`) runs **after** the `await asyncio.sleep(0.01)`.
3. Between those two moments, `delete_user` executes completely (it has no `await` before `users.pop`), removing Charlie from `users`.
4. Application code cannot eliminate this gap because **any I/O operation** (database write, network call, file access) requires `await`, which yields control. Between the yield and the resume, the world can change. The only way to make the check and write atomic is to delegate to a system that supports transactions — like a database engine.

**Task 2c: Instrumented Output:**
```
[create_task] User 3 verified at check time
[delete_user] Deleting user 3 NOW
[create_task] Writing task now. User 3 still exists? False
```

The timeline proves: check passed → user deleted → task written with stale assumption.

**Task 2d: Fixed Code:**

```python
import asyncio

users: dict[int, str] = {1: "Alice", 2: "Bob", 3: "Charlie"}
tasks: dict[int, dict] = {}
next_task_id: int = 1
lock = asyncio.Lock()  # Simulates database-level serialization


async def create_task(title: str, user_id: int) -> dict:
    global next_task_id
    async with lock:  # Check AND write are now atomic
        if user_id not in users:
            raise ValueError(f"User {user_id} does not exist")
        task = {"id": next_task_id, "title": title, "user_id": user_id}
        tasks[next_task_id] = task
        next_task_id += 1
        return task


async def delete_user(user_id: int) -> str:
    async with lock:  # ON DELETE RESTRICT: check references first
        if user_id not in users:
            return f"User {user_id} not found"
        referencing = [t for t in tasks.values() if t["user_id"] == user_id]
        if referencing:
            raise ValueError(
                f"Cannot delete user {user_id}: {len(referencing)} task(s) reference them"
            )
        name = users.pop(user_id)
        return f"Deleted {name}"
```

Now the lock guarantees either: (a) create_task runs first → task created → delete_user sees the reference and raises an error, OR (b) delete_user runs first → user deleted → create_task's check raises ValueError. No interleaving is possible within the locked section.

**Task 2e:**

The `test_no_orphans_under_concurrency` test now passes. After the gather, either the task exists (and so does user 3) or the task does not exist (and user 3 was deleted). The invariant holds.

---

## Solution 2.2: The Counter That Drifted

**Task 2a:**

Phase 2 (reassignment) is where drift begins. `reassign_task` changes the task's `category_id` but does not update either category's `task_count`. After Phase 2:
- Backend: cached=3, actual=2 → drift of +1
- Testing: cached=2, actual=3 → drift of -1

Phase 3 makes it **worse**, not better. `delete_task` reads `tasks[3]["category_id"]` which is now `3` (Testing) due to the reassignment. It decrements Testing's count from 2 to 1. But Testing's actual count goes from 3 to 2 (one task removed). Meanwhile Backend's cached count stays at 3 even though it only has 2 tasks.

After Phase 3:
- Backend: cached=3, actual=2 → **still drifted** (never corrected)
- Testing: cached=1, actual=2 → **still drifted** (decrement applied to wrong starting value)

**Task 2b:**

The compound error: the reassignment left Backend's count inflated (counted a task that left) and Testing's count deflated (didn't count a task that arrived). Then the deletion found the task in Testing (correct *now*) and decremented Testing — but Testing's count was already wrong. The errors don't cancel out; they compound.

This is the **Update Anomaly** applied to cached/derived data. Any code path that modifies the *source data* (`tasks`) without also modifying the *derived data* (`task_count`) creates drift. And in a real system with dozens of code paths (API endpoints, background jobs, admin tools, migration scripts), at least one will forget.

**Task 2d:**

**Fix A (Bandaid):**
```python
def reassign_task(task_id: int, new_category_id: int) -> bool:
    if task_id not in tasks or new_category_id not in categories:
        return False
    old_cat_id = tasks[task_id]["category_id"]
    tasks[task_id]["category_id"] = new_category_id
    categories[old_cat_id]["task_count"] -= 1   # Decrement old
    categories[new_category_id]["task_count"] += 1  # Increment new
    return True
```

This fixes the immediate bug. But every new function that touches tasks must remember to update `task_count`. A bulk import function, a cascade delete, a category merge — each must maintain the cache manually.

**Fix B (Normalization — the correct fix):**
```python
# Remove task_count from categories entirely:
categories = {
    1: {"id": 1, "name": "Backend"},
    2: {"id": 2, "name": "Frontend"},
    3: {"id": 3, "name": "Testing"},
}

def get_task_count(category_id: int) -> int:
    """Always compute from source of truth — like COUNT(*) in SQL."""
    return sum(1 for t in tasks.values() if t["category_id"] == category_id)

def dashboard() -> None:
    print("  Category Dashboard:")
    for cid, cat in categories.items():
        count = get_task_count(cid)
        print(f"    {cat['name']}: {count} tasks")
```

No cache. No drift. The count is always derived from the actual data. This is the database equivalent of `SELECT category_id, COUNT(*) FROM tasks GROUP BY category_id` — the database never "caches" this unless you explicitly ask it to (materialized views). Normalization means "store each fact once and derive everything else."

---

# LEVEL 2.5: EXTEND EXISTING CODE

## The SimpleTable Engine

### Background

Below is a working `Table` class that simulates NOT NULL and UNIQUE constraints in Python. It passes all provided tests. Your job is to **extend** it with three new capabilities — without breaking the existing test suite.

### Existing Code

```python
class IntegrityError(Exception):
    """Raised when a database constraint is violated."""
    pass


class Table:
    def __init__(
        self,
        name: str,
        not_null: list[str] | None = None,
        unique: list[str] | None = None,
    ):
        self.name = name
        self.not_null = not_null or []
        self.unique = unique or []
        self._rows: dict[int, dict] = {}
        self._next_id = 1

    def insert(self, **data: object) -> dict:
        # NOT NULL enforcement
        for col in self.not_null:
            if col not in data or data[col] is None:
                raise IntegrityError(
                    f"NOT NULL violation on '{self.name}.{col}'"
                )

        # UNIQUE enforcement
        for col in self.unique:
            if col in data:
                for row in self._rows.values():
                    if row.get(col) == data[col]:
                        raise IntegrityError(
                            f"UNIQUE violation on '{self.name}.{col}': "
                            f"'{data[col]}' already exists"
                        )

        row_id = self._next_id
        self._next_id += 1
        row = {"id": row_id, **data}
        self._rows[row_id] = row
        return row

    def get(self, row_id: int) -> dict | None:
        return self._rows.get(row_id)

    def find(self, **criteria: object) -> list[dict]:
        return [
            r for r in self._rows.values()
            if all(r.get(k) == v for k, v in criteria.items())
        ]

    def all(self) -> list[dict]:
        return list(self._rows.values())

    def update(self, row_id: int, **data: object) -> bool:
        if row_id not in self._rows:
            return False
        for col in self.unique:
            if col in data:
                for rid, row in self._rows.items():
                    if rid != row_id and row.get(col) == data[col]:
                        raise IntegrityError(
                            f"UNIQUE violation on '{self.name}.{col}': "
                            f"'{data[col]}' already exists"
                        )
        self._rows[row_id].update(data)
        return True

    def delete(self, row_id: int) -> bool:
        if row_id in self._rows:
            del self._rows[row_id]
            return True
        return False
```

### Existing Test Suite (DO NOT MODIFY)

```python
def run_existing_tests() -> None:
    """These tests must ALWAYS pass, before and after your extensions."""

    # Test 1: Insert and retrieve
    t = Table("users", not_null=["name"])
    row = t.insert(name="Alice", email="alice@co.com")
    assert row["id"] == 1
    assert row["name"] == "Alice"
    assert t.get(1) is not None
    assert t.get(999) is None

    # Test 2: NOT NULL enforcement
    t2 = Table("users", not_null=["name", "email"])
    try:
        t2.insert(name="Alice")
        assert False, "Should have raised IntegrityError"
    except IntegrityError:
        pass

    # Test 3: UNIQUE enforcement
    t3 = Table("users", unique=["email"])
    t3.insert(name="Alice", email="a@co.com")
    try:
        t3.insert(name="Bob", email="a@co.com")
        assert False, "Should have raised IntegrityError"
    except IntegrityError:
        pass

    # Test 4: Find with criteria
    t4 = Table("users", not_null=["name"])
    t4.insert(name="Alice", dept="Backend")
    t4.insert(name="Bob", dept="Frontend")
    t4.insert(name="Charlie", dept="Backend")
    assert len(t4.find(dept="Backend")) == 2
    assert len(t4.find(dept="Frontend")) == 1

    # Test 5: Update respects UNIQUE
    t5 = Table("users", unique=["email"])
    t5.insert(name="Alice", email="a@co.com")
    t5.insert(name="Bob", email="b@co.com")
    try:
        t5.update(2, email="a@co.com")
        assert False, "Should have raised IntegrityError"
    except IntegrityError:
        pass

    # Test 6: Delete removes row
    t6 = Table("users", not_null=["name"])
    t6.insert(name="Alice")
    assert len(t6.all()) == 1
    t6.delete(1)
    assert len(t6.all()) == 0

    print("ALL EXISTING TESTS PASSED")


run_existing_tests()
```

---

### Extension A: Foreign Key Validation on Insert

Modify the `Table` constructor to accept a `foreign_keys` parameter that maps column names to parent `Table` instances. When inserting a row, if a foreign key column has a value, that value must be a valid `id` in the parent table. If not, raise `IntegrityError`.

**Your extension must pass these tests:**

```python
def test_extension_a() -> None:
    users = Table("users", not_null=["name"])
    tasks = Table("tasks", not_null=["title"], foreign_keys={"user_id": users})

    alice = users.insert(name="Alice")  # id=1

    # Valid reference
    task = tasks.insert(title="Build API", user_id=1)
    assert task["user_id"] == 1

    # Invalid reference
    try:
        tasks.insert(title="Bad task", user_id=999)
        assert False, "Should have raised IntegrityError"
    except IntegrityError as e:
        assert "foreign key" in str(e).lower()

    # Null FK allowed (task with no owner)
    orphan = tasks.insert(title="Unassigned task")
    assert "user_id" not in orphan or orphan.get("user_id") is None

    print("EXTENSION A TESTS PASSED")
```

---

### Extension B: CHECK Constraints

Modify the `Table` constructor to accept a `checks` parameter: a dict mapping column names to callable validators. On insert AND update, if the column is present, the validator must return `True` or `IntegrityError` is raised.

**Your extension must pass these tests:**

```python
def test_extension_b() -> None:
    tasks = Table(
        "tasks",
        not_null=["title"],
        checks={
            "priority": lambda v: isinstance(v, int) and 1 <= v <= 5,
            "title": lambda v: isinstance(v, str) and len(v) > 0,
        },
    )

    # Valid data
    t1 = tasks.insert(title="Build API", priority=3)
    assert t1["priority"] == 3

    # Invalid priority (too high)
    try:
        tasks.insert(title="Bad", priority=99)
        assert False, "Should have raised IntegrityError"
    except IntegrityError as e:
        assert "check" in str(e).lower()

    # Invalid priority on UPDATE
    try:
        tasks.update(1, priority=0)
        assert False, "Should have raised IntegrityError"
    except IntegrityError:
        pass

    # Valid update
    tasks.update(1, priority=5)
    assert tasks.get(1)["priority"] == 5

    print("EXTENSION B TESTS PASSED")
```

---

### Extension C: RESTRICT on Delete

When a table has foreign keys pointing to a parent table, deleting a row from the parent should **fail** if any child row references it. This simulates `ON DELETE RESTRICT`.

**Hint:** The child table must register itself with the parent table at construction time so the parent knows who depends on it.

**Your extension must pass these tests:**

```python
def test_extension_c() -> None:
    users = Table("users", not_null=["name"])
    tasks = Table("tasks", not_null=["title"], foreign_keys={"user_id": users})

    alice = users.insert(name="Alice")   # id=1
    bob = users.insert(name="Bob")       # id=2
    tasks.insert(title="Build API", user_id=1)  # References Alice

    # Cannot delete Alice — tasks reference her
    try:
        users.delete(1)
        assert False, "Should have raised IntegrityError"
    except IntegrityError as e:
        assert "referenced" in str(e).lower() or "restrict" in str(e).lower()

    # CAN delete Bob — no tasks reference him
    assert users.delete(2) is True
    assert users.get(2) is None

    # Alice still exists (delete was blocked)
    assert users.get(1) is not None

    print("EXTENSION C TESTS PASSED")
```

**After all extensions, run the existing test suite again. It must still pass.**

---

# Level 2.5 Solutions

## Solution: Complete Extended Table Class

```python
class IntegrityError(Exception):
    pass


class Table:
    def __init__(
        self,
        name: str,
        not_null: list[str] | None = None,
        unique: list[str] | None = None,
        foreign_keys: dict[str, "Table"] | None = None,   # Extension A
        checks: dict[str, callable] | None = None,         # Extension B
    ):
        self.name = name
        self.not_null = not_null or []
        self.unique = unique or []
        self.foreign_keys = foreign_keys or {}
        self.checks = checks or {}
        self._rows: dict[int, dict] = {}
        self._next_id = 1
        self._dependents: list[tuple["Table", str]] = []  # Extension C

        # Extension C: register with parent tables
        for fk_col, parent_table in self.foreign_keys.items():
            parent_table._dependents.append((self, fk_col))

    def _validate_checks(self, data: dict) -> None:
        """Extension B: Validate CHECK constraints."""
        for col, check_fn in self.checks.items():
            if col in data:
                if not check_fn(data[col]):
                    raise IntegrityError(
                        f"CHECK violation on '{self.name}.{col}': "
                        f"value '{data[col]}' failed validation"
                    )

    def _validate_foreign_keys(self, data: dict) -> None:
        """Extension A: Validate foreign key references."""
        for fk_col, parent_table in self.foreign_keys.items():
            if fk_col in data and data[fk_col] is not None:
                if parent_table.get(data[fk_col]) is None:
                    raise IntegrityError(
                        f"FOREIGN KEY violation: '{self.name}.{fk_col}'={data[fk_col]} "
                        f"not found in '{parent_table.name}'"
                    )

    def insert(self, **data: object) -> dict:
        for col in self.not_null:
            if col not in data or data[col] is None:
                raise IntegrityError(f"NOT NULL violation on '{self.name}.{col}'")

        for col in self.unique:
            if col in data:
                for row in self._rows.values():
                    if row.get(col) == data[col]:
                        raise IntegrityError(
                            f"UNIQUE violation on '{self.name}.{col}': '{data[col]}' already exists"
                        )

        self._validate_checks(data)          # Extension B
        self._validate_foreign_keys(data)    # Extension A

        row_id = self._next_id
        self._next_id += 1
        row = {"id": row_id, **data}
        self._rows[row_id] = row
        return row

    def get(self, row_id: int) -> dict | None:
        return self._rows.get(row_id)

    def find(self, **criteria: object) -> list[dict]:
        return [
            r for r in self._rows.values()
            if all(r.get(k) == v for k, v in criteria.items())
        ]

    def all(self) -> list[dict]:
        return list(self._rows.values())

    def update(self, row_id: int, **data: object) -> bool:
        if row_id not in self._rows:
            return False
        for col in self.unique:
            if col in data:
                for rid, row in self._rows.items():
                    if rid != row_id and row.get(col) == data[col]:
                        raise IntegrityError(
                            f"UNIQUE violation on '{self.name}.{col}': '{data[col]}' already exists"
                        )
        self._validate_checks(data)  # Extension B: also on update
        self._rows[row_id].update(data)
        return True

    def delete(self, row_id: int) -> bool:
        # Extension C: RESTRICT — block if referenced
        for child_table, fk_col in self._dependents:
            referencing = child_table.find(**{fk_col: row_id})
            if referencing:
                raise IntegrityError(
                    f"RESTRICT: Cannot delete '{self.name}' id={row_id}: "
                    f"referenced by {len(referencing)} row(s) in '{child_table.name}'"
                )

        if row_id in self._rows:
            del self._rows[row_id]
            return True
        return False
```

**Why this matters:**

Extension A teaches that foreign keys are a **write-time validation** — the engine checks at the moment you insert, not when you query. Extension B teaches that CHECK constraints are the storage-layer equivalent of Pydantic's `Field(gt=0)`. Extension C is the most revealing: it shows that referential integrity is **bidirectional**. The parent must know about its children. In a real database, this is handled automatically by the engine when you declare `REFERENCES parent(id)`. In Python, you had to build the registration mechanism yourself, which makes the cost and complexity visible.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## The Concert Hall Booking System

### Scenario

You are building a ticket booking system for a 50-seat concert. During the sale window, 80 users will attempt to book tickets simultaneously.

**Constraints (the naive approach cannot satisfy all of these):**
- **No overselling:** Exactly 50 tickets exist. If 80 users try to book concurrently, exactly 50 succeed and exactly 30 fail. Never 51 successes. Never 49.
- **No double-booking:** A user cannot hold more than one ticket per booking call.
- **Atomicity for group bookings:** A user can request multiple tickets (e.g., 3). Either ALL 3 are assigned or NONE are. No partial allocations.
- **All operations are async:** Booking involves simulated I/O (`await asyncio.sleep`) for payment processing.

### Provided Skeleton

```python
import asyncio


class TicketSystem:
    def __init__(self, total_tickets: int):
        self.total_tickets = total_tickets
        self.available_tickets = total_tickets
        self.bookings: dict[str, list[int]] = {}  # user_id -> [ticket_numbers]
        self._next_ticket = 1

    async def book_ticket(self, user_id: str) -> dict:
        """Book a single ticket.

        Returns:
            {"success": True,  "ticket_number": int, "reason": "Booked"}
            {"success": False, "ticket_number": None, "reason": "..."}

        Rules:
        - Same user cannot book twice
        - Cannot oversell
        - Must include `await asyncio.sleep(0.001)` to simulate payment I/O
        """
        raise NotImplementedError("Implement in 3a")

    async def book_tickets(self, user_id: str, count: int) -> dict:
        """Book multiple tickets atomically.

        Returns:
            {"success": True,  "ticket_numbers": [int, ...], "reason": "Booked"}
            {"success": False, "ticket_numbers": [],          "reason": "..."}

        Rules:
        - ALL tickets succeed or NONE do (atomicity)
        - Must include `await asyncio.sleep(0.001)` to simulate payment I/O
        """
        raise NotImplementedError("Implement in 3c")
```

### Provided Tests

```python
# ─── TEST 3a: Sequential Correctness ───

def test_basic_booking():
    system = TicketSystem(total_tickets=10)
    result = asyncio.run(system.book_ticket(user_id="alice"))
    assert result["success"] is True
    assert result["ticket_number"] is not None
    assert system.available_tickets == 9
    print("  test_basic_booking PASSED")

def test_duplicate_rejected():
    system = TicketSystem(total_tickets=10)
    asyncio.run(system.book_ticket(user_id="alice"))
    result = asyncio.run(system.book_ticket(user_id="alice"))
    assert result["success"] is False
    assert "already" in result["reason"].lower()
    assert system.available_tickets == 9
    print("  test_duplicate_rejected PASSED")

def test_sold_out():
    system = TicketSystem(total_tickets=2)
    asyncio.run(system.book_ticket(user_id="alice"))
    asyncio.run(system.book_ticket(user_id="bob"))
    result = asyncio.run(system.book_ticket(user_id="charlie"))
    assert result["success"] is False
    assert system.available_tickets == 0
    print("  test_sold_out PASSED")


# ─── TEST 3b: Concurrent Safety ───

def test_concurrent_no_overselling():
    system = TicketSystem(total_tickets=50)

    async def rush():
        coros = [system.book_ticket(user_id=f"user_{i}") for i in range(80)]
        return await asyncio.gather(*coros)

    results = asyncio.run(rush())
    successes = [r for r in results if r["success"]]
    failures = [r for r in results if not r["success"]]

    assert len(successes) == 50, f"Expected 50 successes, got {len(successes)}"
    assert len(failures) == 30, f"Expected 30 failures, got {len(failures)}"
    assert system.available_tickets == 0

    # No duplicate ticket numbers
    ticket_nums = [r["ticket_number"] for r in successes]
    assert len(set(ticket_nums)) == 50, "Duplicate ticket numbers detected!"
    print("  test_concurrent_no_overselling PASSED")


# ─── TEST 3c: Atomic Multi-Ticket ───

def test_atomic_group_booking():
    system = TicketSystem(total_tickets=5)

    # Book 3 tickets (should succeed)
    r1 = asyncio.run(system.book_tickets(user_id="group_a", count=3))
    assert r1["success"] is True
    assert len(r1["ticket_numbers"]) == 3
    assert system.available_tickets == 2

    # Try to book 3 more (should FAIL — only 2 left)
    r2 = asyncio.run(system.book_tickets(user_id="group_b", count=3))
    assert r2["success"] is False
    assert system.available_tickets == 2, (
        f"ATOMICITY VIOLATION: available={system.available_tickets}, expected 2. "
        f"Tickets were partially deducted!"
    )
    print("  test_atomic_group_booking PASSED")
```

---

### 3a: Implement Sequential Booking

Implement `book_ticket` so that `test_basic_booking`, `test_duplicate_rejected`, and `test_sold_out` pass. You **must** include `await asyncio.sleep(0.001)` somewhere in the method to simulate payment I/O (this is required for the concurrency tests later).

### 3b: Make It Concurrent-Safe

Run `test_concurrent_no_overselling`. Your 3a implementation will almost certainly fail this test (80 successes instead of 50). Fix `book_ticket` so that concurrent access cannot oversell. The same three sequential tests must still pass.

### 3c: Implement Atomic Multi-Ticket Booking

Implement `book_tickets` so that `test_atomic_group_booking` passes. The critical constraint: if a group requests 3 tickets but only 2 remain, **zero** tickets are deducted. Not 2. Not 1. Zero. All-or-nothing.

---

# Level 3 Solutions

## Solution 3a: Naive Sequential (passes 3a tests, fails 3b)

```python
async def book_ticket(self, user_id: str) -> dict:
    if user_id in self.bookings:
        return {"success": False, "ticket_number": None, "reason": "Already booked"}

    if self.available_tickets <= 0:
        return {"success": False, "ticket_number": None, "reason": "Sold out"}

    await asyncio.sleep(0.001)  # Simulate payment I/O

    ticket = self._next_ticket
    self._next_ticket += 1
    self.available_tickets -= 1
    self.bookings[user_id] = [ticket]
    return {"success": True, "ticket_number": ticket, "reason": "Booked"}
```

This fails `test_concurrent_no_overselling`: all 80 coroutines read `available_tickets = 50` before the sleep, all pass the check, all decrement after waking. Result: 80 successes, `available_tickets = -30`.

## Solution 3b: Concurrent-Safe (passes all 3a and 3b tests)

```python
def __init__(self, total_tickets: int):
    # ... (same as before)
    self._lock = asyncio.Lock()

async def book_ticket(self, user_id: str) -> dict:
    async with self._lock:
        if user_id in self.bookings:
            return {"success": False, "ticket_number": None, "reason": "Already booked"}

        if self.available_tickets <= 0:
            return {"success": False, "ticket_number": None, "reason": "Sold out"}

        await asyncio.sleep(0.001)

        ticket = self._next_ticket
        self._next_ticket += 1
        self.available_tickets -= 1
        self.bookings[user_id] = [ticket]
        return {"success": True, "ticket_number": ticket, "reason": "Booked"}
```

The lock serializes access. Only one coroutine at a time can execute the check-sleep-write sequence. The first 50 succeed; the remaining 30 see `available_tickets <= 0` and are rejected.

This is `ISOLATION`: each transaction behaves as if it's the only one running.

## Solution 3c: Atomic Multi-Ticket

```python
async def book_tickets(self, user_id: str, count: int) -> dict:
    async with self._lock:
        # CHECK FIRST — before any state mutation
        if self.available_tickets < count:
            return {
                "success": False,
                "ticket_numbers": [],
                "reason": f"Need {count}, only {self.available_tickets} available",
            }

        await asyncio.sleep(0.001)

        # Only reached if ALL tickets are available
        tickets = []
        for _ in range(count):
            tickets.append(self._next_ticket)
            self._next_ticket += 1
            self.available_tickets -= 1

        self.bookings[user_id] = tickets
        return {"success": True, "ticket_numbers": tickets, "reason": "Booked"}
```

The key insight: **check availability for the entire group BEFORE modifying any state.** A naive loop that checks and deducts one-at-a-time would leave the system in an inconsistent state if it runs out partway through (2 tickets deducted from a failed 3-ticket request).

This is `ATOMICITY`: the entire multi-ticket operation succeeds or fails as one indivisible unit. It mirrors how a database `BEGIN ... COMMIT` / `ROLLBACK` works — either all statements in the transaction commit, or all are rolled back.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## When Third Normal Form Backfires

**Task:** Describe a realistic scenario where normalizing to 3NF causes *more* problems than storing denormalized data. Then write a Python snippet that demonstrates the performance cost of the normalized approach.

**Requirements:**
- The scenario must be specific (not just "sometimes denormalization is faster").
- You must show a concrete measurement (timing) proving the normalized approach is slower.
- You must explain what *specifically* makes this scenario different from the cases where normalization is correct.
- You must state the **tradeoff**: what do you lose by denormalizing?

---

# Level 4 Solution

**Scenario: Real-Time Analytics Dashboard**

A dashboard that refreshes every 5 seconds shows: for each project, the project name, the owner's name, the category name, the number of tasks, and the average priority. With full normalization, every refresh requires joining 4 tables and computing aggregates — multiplied by 100 concurrent users watching the dashboard.

```python
import time

# ─── Normalized (4 "tables") ───
users = {i: {"id": i, "name": f"User_{i}"} for i in range(1, 101)}
categories = {i: {"id": i, "name": f"Category_{i}"} for i in range(1, 21)}
projects = {
    i: {"id": i, "name": f"Project_{i}", "owner_id": (i % 100) + 1, "category_id": (i % 20) + 1}
    for i in range(1, 501)
}
tasks_table = {
    i: {"id": i, "project_id": (i % 500) + 1, "priority": (i % 5) + 1}
    for i in range(1, 5001)
}


def render_dashboard_normalized() -> list[dict]:
    result = []
    for pid, proj in projects.items():
        owner = users[proj["owner_id"]]
        cat = categories[proj["category_id"]]
        proj_tasks = [t for t in tasks_table.values() if t["project_id"] == pid]
        avg_pri = sum(t["priority"] for t in proj_tasks) / len(proj_tasks) if proj_tasks else 0
        result.append({
            "project": proj["name"], "owner": owner["name"],
            "category": cat["name"], "task_count": len(proj_tasks), "avg_priority": avg_pri,
        })
    return result


# ─── Denormalized (precomputed "materialized view") ───
dashboard_cache: list[dict] = []

def rebuild_cache() -> None:
    global dashboard_cache
    dashboard_cache = render_dashboard_normalized()  # Computed once on write

def render_dashboard_denormalized() -> list[dict]:
    return dashboard_cache  # O(1) lookup


# ─── Timing ───
rebuild_cache()

start = time.perf_counter()
for _ in range(100):  # 100 dashboard refreshes
    render_dashboard_normalized()
norm_time = time.perf_counter() - start

start = time.perf_counter()
for _ in range(100):
    render_dashboard_denormalized()
denorm_time = time.perf_counter() - start

print(f"Normalized:   {norm_time:.4f}s for 100 renders")
print(f"Denormalized: {denorm_time:.4f}s for 100 renders")
print(f"Speedup: {norm_time / denorm_time:.0f}x")
```

**Why this scenario is different:** The dashboard is **read-heavy** (100 users × refresh every 5 seconds = 20 reads/sec) and **write-light** (tasks are created/updated a few times per minute). The aggregated data changes rarely. Recomputing 4-table joins and aggregates on every read wastes resources.

**The tradeoff:** The denormalized cache can go stale. If a task is created but the cache isn't rebuilt, the dashboard shows the old count. You gain read performance but lose real-time accuracy. The lecture's rule applies: "Start normalized. Denormalize only when you have MEASURED a performance problem." This scenario qualifies — the measurement proves the cost.

---

# LEVEL 5: THE TEACH BACK

## "Foreign Keys Are Redundant"

**Scenario:**

A junior developer on your team submits a pull request for a new task creation endpoint. In the PR description, they write:

> "I didn't add a foreign key constraint for `user_id` in the tasks table. I already validate that the user exists in my endpoint code using Pydantic and a database query before creating the task. Adding a foreign key would be redundant validation that just slows down inserts. Plus, foreign keys make it annoying to delete users — you have to handle all the tasks first."

**Task:** Write a detailed code review response addressing each of the four claims:
1. "I already validate in my endpoint code" — why this isn't sufficient
2. "It would be redundant" — why layered validation is a feature, not a bug
3. "It slows down inserts" — what the actual cost/benefit tradeoff is
4. "It makes deleting users annoying" — why this "annoyance" is actually protecting you

Use specific examples from the exercises you completed in this set. Reference at least one scenario where application-level validation failed.

---

# Level 5 Solution

**Model Code Review Response:**

---

Hey, thanks for the PR. The endpoint logic looks solid, but I need to push back on the FK decision. Let me address each point.

**1. "I already validate in my endpoint code."**

Your endpoint checks `user_exists = await db.get_user(user_id)`. That works for requests that come through this endpoint. But our system has other write paths: the background job in `tasks/worker.py` that creates tasks from queued events, the admin bulk-import script in `scripts/import_tasks.py`, and the migration we're planning for next sprint. Every one of those code paths must independently remember to validate user_id. If any developer on any path forgets — or if we add a new path six months from now — we get orphaned tasks with no error.

In Exercise 2.1 of our training set, we saw exactly this: an application-level check passed at T1, but by the time the task was written at T2, the user had been deleted by a concurrent request. The check and the write weren't atomic. A foreign key constraint IS atomic — the database checks the reference at the exact moment of the write, inside the transaction.

**2. "It would be redundant."**

Redundant validation across layers is called "defense in depth," and it's the same principle we use for Pydantic + database types. Pydantic catches bad input at the API boundary with nice error messages. The foreign key catches invalid references at the storage boundary as the last line of defense. If your code has a bug that bypasses Pydantic (and bugs happen), the FK is your safety net. Exercise 2.2 demonstrated this with cached counts: the "fast path" code (dashboard cache) bypassed the validation logic, creating data drift. The storage layer is the only layer that can never be bypassed.

**3. "It slows down inserts."**

A foreign key check is an index lookup on the referenced table's primary key. On a B-tree index, this is O(log n) — microseconds for millions of rows. The `await asyncio.sleep(0.001)` we use to simulate network I/O in our exercises takes 1000 microseconds. The FK check takes roughly 0.01–0.1 microseconds. You'd need to insert thousands of rows per second before the FK overhead is even measurable. What IS expensive is debugging orphaned data in production three months from now. The cost comparison isn't "FK check time vs. no check time." It's "microseconds of prevention vs. hours of incident response."

**4. "It makes deleting users annoying."**

That "annoyance" is the system telling you: "Hey, 47 tasks reference this user. What should happen to them?" Without the FK, you'd delete the user, the 47 tasks would silently become orphans, and you'd discover it when a customer reports that their task list shows `[DELETED USER]` — or worse, as in Exercise 1.2, when a new user reuses the ID and the tasks *silently appear to belong to someone else entirely.*

With the FK, you're forced to make an explicit decision: `ON DELETE CASCADE` (delete the tasks too), `ON DELETE SET NULL` (remove the assignment), or handle it in application logic before the delete. Each choice is a deliberate product decision, not an accidental data corruption.

Please add the foreign key. I'll approve once it's in.

---