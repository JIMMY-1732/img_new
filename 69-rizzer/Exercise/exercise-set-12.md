# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "API design is about picking the right endpoint names and status codes. Pagination is just slicing a list. PATCH just skips empty fields. 403 is more accurate than 404 when a user isn't allowed in. It all sounds straightforward."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---|---|
| 1 | Offset and cursor pagination produce the same data correctness | Concurrent inserts/deletes shift the offset window — cursor stability is impossible to replicate with offsets |
| 2 | One Pydantic model for input and output is clean DRY code | Mass assignment lets clients write to server-controlled fields (`owner_id`, `is_admin`) with no error raised |
| 3 | `model_dump()` + `if value is not None` correctly implements PATCH | It cannot distinguish "client did not send this field" from "client explicitly sent null" — nullable fields can never be cleared |
| 4 | `403 Forbidden` is the correct status when a user requests another user's resource | 403 confirms the resource *exists* — an attacker can enumerate the entire ID space by watching for 403 vs 404 |
| 5 | Filtering can run anywhere in the pipeline as long as it runs before the response | Ownership filter must be step one; pagination must be last — any other order is both a security hole and a correctness bug |
| 6 | An ETag header on the 200 response is optional performance optimization | Without ETag on 200, the client has no cache key — conditional requests can never work regardless of how many times you implement the 304 path |

## The Main Purpose (Essence)

**Core Insight:** Every API design principle in this lecture exists to protect a specific party from a specific failure mode. Pagination protects the server from memory exhaustion. Cursor stability protects the consumer from missed or duplicated items. Input/output separation protects the data model from client-controlled writes to internal fields. 404-not-403 protects the system from information leakage. Ownership-first filtering protects both correctness and performance — filtering client data before ownership means doing expensive work on data you have no right to return.

**The Key Distinctions:**

- `model_dump()` → "serialize every field, including defaults" — dangerous in PATCH
- `model_dump(exclude_unset=True)` → "serialize only what the client actually sent" — correct in PATCH
- Offset pagination → position is a number, shifts when data changes
- Cursor pagination → position is a value anchored to an item identity, stable under concurrent writes
- `403 Forbidden` → "resource exists, you can't have it"
- `404 Not Found` → "nothing here" — reveals nothing about ownership

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Phantom Page

```python
# Simulating a paginated feed with a concurrent write between page requests.
# The dataset is ordered newest-first (highest ID first).

POSTS = [{"id": i, "title": f"Post {i}"} for i in range(10, 0, -1)]
# Initial state: [{id:10}, {id:9}, {id:8}, {id:7}, {id:6},
#                 {id:5},  {id:4}, {id:3}, {id:2}, {id:1}]

PAGE_SIZE = 5


def fetch_page(offset: int) -> list[dict]:
    return POSTS[offset : offset + PAGE_SIZE]


# ── Client fetches page 1 ───────────────────────────────────────────────
page_1 = fetch_page(offset=0)
page_1_ids = [p["id"] for p in page_1]

# ── A new post is published between page requests ────────────────────────
POSTS.insert(0, {"id": 11, "title": "Breaking News"})  # Prepended as newest

# ── Client fetches page 2 ───────────────────────────────────────────────
page_2 = fetch_page(offset=PAGE_SIZE)
page_2_ids = [p["id"] for p in page_2]

# ── Diagnosis ────────────────────────────────────────────────────────────
original_ids = set(range(1, 11))
all_seen = set(page_1_ids) | set(page_2_ids)

print(f"Page 1 IDs:   {page_1_ids}")
print(f"Page 2 IDs:   {page_2_ids}")
print(f"Seen in both: {set(page_1_ids) & set(page_2_ids)}")
print(f"Never seen:   {original_ids - all_seen}")
```

**Questions:**

1. **Predict the output.** Write the exact value of each `print` statement.

2. **Explain why.** Post ID 1 existed before any requests were made. Why did the client never receive it? What caused the other anomaly in the results?

3. **Mental Trace.** Draw two snapshots of `POSTS`: one before the insert and one after. Annotate which index positions each `fetch_page(offset)` call maps to. Show explicitly why the two windows overlap and why one element falls off the end.

---

## Exercise 1.2: The Null Paradox

```python
from dataclasses import dataclass
from typing import Optional


@dataclass
class NoteUpdate:
    """Fields a client can include in a PATCH /notes/{id} request."""
    title: Optional[str] = None
    body: Optional[str] = None    # Nullable — client may intentionally clear it
    pinned: Optional[bool] = None


def apply_patch(stored: dict, update: NoteUpdate) -> dict:
    """Apply changed fields to the stored note."""
    result = stored.copy()
    candidates = {
        "title": update.title,
        "body": update.body,
        "pinned": update.pinned,
    }
    for field, value in candidates.items():
        if value is not None:        # ← "Only apply non-null values"
            result[field] = value
    return result


stored_note = {
    "id": 7,
    "title": "Grocery list",
    "body": "Milk, eggs, bread",
    "pinned": True,
}

# ── Scenario A: Unpin the note (set pinned to False) ─────────────────────
update_a = NoteUpdate(pinned=False)
result_a = apply_patch(stored_note, update_a)
print(f"A — pinned: {result_a['pinned']!r}")
print(f"A — body:   {result_a['body']!r}")

# ── Scenario B: Client sends PATCH {"body": null} to clear the body ───────
update_b = NoteUpdate(body=None)    # Client explicitly sent {"body": null}
result_b = apply_patch(stored_note, update_b)
print(f"B — body cleared? {result_b['body'] is None}")
print(f"B — body value:   {result_b['body']!r}")
```

**Questions:**

1. **Predict the output.** Write the exact value of all four `print` statements.

2. **Explain the paradox.** The client for Scenario B sent `{"body": null}`. The intent is explicit: clear the body field. Why does the `if value is not None` guard fail to honour this intent?

3. **Identify the ambiguity.** In the `candidates` dict, `"body": None` can arise from two completely different client actions. Name both actions and explain why the function cannot distinguish between them at this point in the code.

---

## Exercise 1.3: The Helpful 403

```python
from dataclasses import dataclass


@dataclass
class APIResponse:
    status: int
    body: str

    def __repr__(self) -> str:
        return f"HTTP {self.status}: {self.body!r}"


# Simulated private notes database
NOTES = {
    101: {"id": 101, "owner_id": 1, "title": "Alice's budget"},
    102: {"id": 102, "owner_id": 2, "title": "Bob's roadmap"},
    103: {"id": 103, "owner_id": 1, "title": "Alice's diary"},
}


def get_note_v1(note_id: int, user_id: int) -> APIResponse:
    """Distinguishes 'does not exist' from 'not your note'."""
    if note_id not in NOTES:
        return APIResponse(404, "Not found")
    if NOTES[note_id]["owner_id"] != user_id:
        return APIResponse(403, "Forbidden")
    return APIResponse(200, NOTES[note_id]["title"])


def get_note_v2(note_id: int, user_id: int) -> APIResponse:
    """Treats 'not your note' the same as 'does not exist'."""
    if note_id not in NOTES:
        return APIResponse(404, "Not found")
    if NOTES[note_id]["owner_id"] != user_id:
        return APIResponse(404, "Not found")
    return APIResponse(200, NOTES[note_id]["title"])


# An attacker (user_id=99) probes for existing note IDs
print("=== V1 (403 approach) ===")
for nid in range(99, 106):
    print(f"  GET /notes/{nid} → {get_note_v1(nid, user_id=99)}")

print("\n=== V2 (404 approach) ===")
for nid in range(99, 106):
    print(f"  GET /notes/{nid} → {get_note_v2(nid, user_id=99)}")
```

**Questions:**

1. **Predict the output** of both loops, line by line.

2. **Explain the attack.** After running the V1 loop, what has the attacker learned about the database that they did not know before probing? What does the V2 loop reveal by comparison?

3. **Quantify the exposure.** Suppose the system uses sequential integer IDs from 1 to 1,000,000. Using V1, how many HTTP requests does an attacker need to build a complete map of all existing note IDs? What does switching to V2 change about that number, and what does it change about the *information* gained per request?

---

# LEVEL 1 SOLUTIONS

## Solution 1.1: The Phantom Page

**1. Predicted output:**
```
Page 1 IDs:   [10, 9, 8, 7, 6]
Page 2 IDs:   [6, 5, 4, 3, 2]
Seen in both: {6}
Never seen:   {1}
```

**2. Why:**
Before the insert, `POSTS` has 10 elements: indices 0–9 map to IDs 10–1. Page 1 reads `[0:5]` and gets IDs 10 through 6. After `POSTS.insert(0, {id:11})`, every existing element shifts right by one — what was at index 5 is now at index 6. Page 2 reads `[5:10]` of the *new* 11-element list. Index 5 is now ID 6 — already seen on page 1. ID 1, previously at index 9, is now at index 10, outside the 10-element page-2 window entirely.

**3. Diagram:**

Before insert:
```
Index:  [0]  [1]  [2]  [3]  [4] │ [5]  [6]  [7]  [8]  [9]
ID:      10    9    8    7    6  │   5    4    3    2    1
         [────────PAGE 1────────]  [────────PAGE 2────────]
```

After `POSTS.insert(0, {id:11})`:
```
Index:  [0]  [1]  [2]  [3]  [4]  [5] │ [6]  [7]  [8]  [9]  [10]
ID:      11   10    9    8    7    6  │   5    4    3    2     1
                                       [────────PAGE 2─────────]
```

Page 2 now starts at index 5 in the new array, landing on ID 6 — already delivered. ID 1 is at index 10, unreachable by a window of size 5 starting at offset 5.

**Key Insight:** Offset pagination locates items by *position number*, not by *item identity*. Any insertion before the offset position shifts everything. Cursor pagination would have said "give me items after ID 6" — a new prepend at ID 11 does not move ID 6's position in the sorted-by-identity sequence.

---

## Solution 1.2: The Null Paradox

**1. Predicted output:**
```
A — pinned: False
A — body:   'Milk, eggs, bread'
B — body cleared? False
B — body value:   'Milk, eggs, bread'
```

**2. Why:**
In Scenario A, `update.pinned` is `False`. The guard `if value is not None` evaluates `False is not None` → `True`. The field is correctly applied.

In Scenario B, the client sent `{"body": null}`, meaning "set body to nothing." But `NoteUpdate(body=None)` produces the same Python dataclass as `NoteUpdate()` — both have `body=None` as the attribute value. The guard cannot distinguish the client's intent from the default. It sees `None`, concludes "nothing to apply," and leaves `stored_note["body"]` unchanged at `"Milk, eggs, bread"`. The clear instruction was silently dropped.

**3. The ambiguity:**
In `candidates`, `"body": None` represents two distinct client actions:
- **"Client did not send the `body` field"** — `NoteUpdate()` or `NoteUpdate(title="x")`, where `body` defaults to `None`
- **"Client explicitly sent `{"body": null}`"** — `NoteUpdate(body=None)`, deserialized from a JSON body that includes the key

Both produce an identical Python object. At the level of a plain dataclass with `Optional[str] = None`, there is no sentinel that records whether the caller *passed* the argument. The `None` value erases all history of how it got there.

**The Fix:** Pydantic's `model_fields_set` tracks which fields were present in the input:
```python
from pydantic import BaseModel

class NoteUpdate(BaseModel):
    title: str | None = None
    body: str | None = None
    pinned: bool | None = None

def apply_patch(stored: dict, update: NoteUpdate) -> dict:
    result = stored.copy()
    # Only includes fields the client actually sent in the JSON body
    result.update(update.model_dump(exclude_unset=True))
    return result
```
`{"body": null}` → `model_fields_set = {"body"}` → `model_dump(exclude_unset=True)` returns `{"body": None}` → the clear is applied. Omitting `body` entirely → it does not appear in the dump → the stored value is untouched.

---

## Solution 1.3: The Helpful 403

**1. Predicted output:**
```
=== V1 (403 approach) ===
  GET /notes/99 → HTTP 404: 'Not found'
  GET /notes/100 → HTTP 404: 'Not found'
  GET /notes/101 → HTTP 403: 'Forbidden'
  GET /notes/102 → HTTP 403: 'Forbidden'
  GET /notes/103 → HTTP 403: 'Forbidden'
  GET /notes/104 → HTTP 404: 'Not found'
  GET /notes/105 → HTTP 404: 'Not found'

=== V2 (404 approach) ===
  GET /notes/99 → HTTP 404: 'Not found'
  GET /notes/100 → HTTP 404: 'Not found'
  GET /notes/101 → HTTP 404: 'Not found'
  GET /notes/102 → HTTP 404: 'Not found'
  GET /notes/103 → HTTP 404: 'Not found'
  GET /notes/104 → HTTP 404: 'Not found'
  GET /notes/105 → HTTP 404: 'Not found'
```

**2. The attack:**
After the V1 loop, the attacker knows — without accessing any content — that note IDs 101, 102, and 103 **exist** in the database (they got 403, not 404). IDs 99, 100, 104, and 105 do not exist. The attacker now holds a partial inventory of real resource IDs.

The V2 loop reveals nothing. Every response is identical: HTTP 404. The attacker cannot distinguish "real note, wrong owner" from "ID never existed." Zero information is leaked per request, regardless of the actual database state.

**3. Quantified exposure:**
With V1: The attacker issues 1,000,000 sequential requests. 403 responses mark real IDs; 404 responses mark gaps. The attacker receives a complete, accurate map of every existing note ID in O(N) requests. The scan is cheap — no credentials, no sophisticated technique, just a loop.

V2 does not reduce the number of requests. It eliminates the *information content* of each response: every response is 404, so the attacker's result is 1,000,000 identical responses. The map is uncomputable. Switching from V1 to V2 changes the per-request information yield from one bit (exists/doesn't exist) to zero bits.

---

# LEVEL 2: DEBUG, FIX & VERIFY

## Exercise 2.1: The Generous Model

**Context:** A Task Manager API was shipped quickly. All existing tests pass and the endpoints respond correctly. A security audit is scheduled. You have 30 minutes to find and close the vulnerability before the auditor does.

```python
# task_api.py
from fastapi import FastAPI
from fastapi.testclient import TestClient
from pydantic import BaseModel
from typing import Optional

app = FastAPI()


class Task(BaseModel):
    """One model used for creation, update, and all responses."""
    title: str
    description: Optional[str] = None
    priority: int = 1
    status: str = "pending"
    owner_id: Optional[int] = None       # Should be set by the auth system
    is_admin_task: bool = False          # Should be admin-only


TASKS: list[dict] = []
_next_id = 1


def _current_user_id() -> int:
    """Auth stub — Week 9 replaces this with JWT extraction."""
    return 42


@app.post("/v1/tasks", status_code=201)
def create_task(task: Task):
    global _next_id
    new = task.model_dump()
    new["id"] = _next_id
    new["owner_id"] = _current_user_id()   # ← Explicitly overridden
    _next_id += 1
    TASKS.append(new)
    return new
```

---

**Task 2a: Identify the Flaw**

The handler explicitly overrides `owner_id`. This protects one field. But there is another field in the `Task` model that a client can set arbitrarily — and the handler does nothing to prevent it. Name that field and explain why accepting it from a client is a security problem. Do NOT write any fix yet.

---

**Task 2b: Explain the Mechanism**

1. Pydantic validates that the attacker's extra field is a valid type. It is. Why does Pydantic not reject the request?
2. What is "mass assignment" in this context? Where does the vulnerability originate — in the validation library, in the framework, or in the model design?
3. The handler overrides `owner_id` explicitly. Does this mean a developer could "protect" all sensitive fields this way instead of creating separate models? What is the practical problem with that approach?

---

**Task 2c: Instrument to Prove Diagnosis**

Write a test using `TestClient` that demonstrates the vulnerability. The test must:
- POST a task with the sensitive field set to a non-default value
- Assert that the stored response contains the attacker's value (not the default)
- This test should **PASS on the current broken code** (proving the exploit works)

```python
client = TestClient(app)

def test_attacker_can_set_privileged_field():
    r = client.post("/v1/tasks", json={
        "title": "Normal-looking task",
        # YOUR CODE: send the field that should be server-only
    })
    assert r.status_code == 201
    # YOUR CODE: assertion proving the exploit succeeded
```

Show the test output.

---

**Task 2d: Fix the Code**

Redesign the models using input/output separation:

1. `TaskCreate` — the only fields a client may provide at creation time
2. `TaskResponse` — the full shape the API always returns (including server-set fields)
3. Update `create_task` to use `TaskCreate` as input and `TaskResponse` as `response_model`

Every server-set field (`status`, `owner_id`, `is_admin_task`) must be enforced in the handler, not in the model's default.

---

**Task 2e: Write a Verification Test**

Write a test that:
- **FAILS** on the original single-model code (the exploit succeeds)
- **PASSES** on your fixed code (the server ignores the attacker's field)

```python
def test_client_cannot_set_privileged_field():
    r = client.post("/v1/tasks", json={
        "title": "Sneaky task",
        "is_admin_task": True,   # ← Attempt to escalate
    })
    assert r.status_code == 201
    # YOUR ASSERTION: regardless of what the client sent,
    # is_admin_task in the response must be False
```

---

## Exercise 2.2: The Backwards Filter

**Context:** You are reviewing a PR. The developer says "I tested it — filtering works great." You run the endpoint against a slightly larger dataset and something feels wrong. The numbers don't add up.

```python
# notes_api.py
from fastapi import FastAPI, Query
from fastapi.testclient import TestClient

app = FastAPI()

# 20 notes alternating active/archived
NOTES = [
    {"id": i, "title": f"Note {i}", "status": "active" if i % 2 == 1 else "archived"}
    for i in range(1, 21)
]
# Active  (odd IDs):  1, 3, 5, 7, 9, 11, 13, 15, 17, 19  — 10 total
# Archived (even IDs): 2, 4, 6, 8, 10, 12, 14, 16, 18, 20 — 10 total


@app.get("/notes")
def list_notes(
    status: str | None = Query(default=None),
    limit: int = Query(default=5, ge=1),
    offset: int = Query(default=0, ge=0),
):
    page = NOTES[offset : offset + limit]                   # ← Step 1: Paginate

    if status is not None:
        page = [n for n in page if n["status"] == status]  # ← Step 2: Filter

    return {"items": page, "returned": len(page), "requested": limit}


client = TestClient(app)
```

---

**Task 2a: Identify the Flaw**

Run these two requests mentally (or actually):

```python
r1 = client.get("/notes?status=active&limit=5&offset=0")
print(f"With filter — returned: {r1.json()['returned']} of {r1.json()['requested']} requested")
print(f"IDs: {[n['id'] for n in r1.json()['items']]}")

r2 = client.get("/notes?limit=5&offset=0")
print(f"\nNo filter — returned: {r2.json()['returned']}")
```

The second request is correct. The first is not. Identify the exact line in `list_notes` that causes the mismatch, and state what invariant it violates.

---

**Task 2b: Explain the Mechanism**

Trace through `list_notes` step by step when called with `status="active"`, `limit=5`, `offset=0`. Write out which notes are present after each operation. Explain why this violates the API contract implied by the query parameters — what does the client believe they are asking for, and what does the server actually deliver?

---

**Task 2c: Instrument to Prove Diagnosis**

Add diagnostic `print` statements to `list_notes` to expose the intermediate state:

```python
def list_notes_instrumented(...):
    page = NOTES[offset : offset + limit]
    print(f"After pagination: {len(page)} items — IDs: {[n['id'] for n in page]}")

    if status is not None:
        page = [n for n in page if n["status"] == status]
        print(f"After filter:     {len(page)} items — IDs: {[n['id'] for n in page]}")

    return {"items": page, "returned": len(page), "requested": limit}
```

Show the printed output for `status="active"`, `limit=5`, `offset=0`. Use the output to explain, concretely, why only 2–3 items were returned instead of 5.

---

**Task 2d: Fix the Code**

Fix `list_notes` so filtering always precedes pagination. Add a `total_matching` field to the response representing how many items match the filter before any pagination is applied.

---

**Task 2e: Write a Verification Test**

Write a test that:
- **FAILS** on the original paginate-then-filter implementation
- **PASSES** on your corrected filter-then-paginate implementation

```python
def test_filter_returns_full_page():
    """
    A dataset with 10 active notes.
    Requesting limit=5 must return exactly 5 active notes.
    Returning fewer means pagination ran before filtering.
    """
    r = client.get("/notes?status=active&limit=5&offset=0")
    data = r.json()
    assert data["returned"] == ???          # YOUR VALUE
    assert data["total_matching"] == ???    # YOUR VALUE
    assert all(n["status"] == "active" for n in data["items"])
```

---

# LEVEL 2 SOLUTIONS

## Solution 2.1: The Generous Model

**Task 2a: Identify the Flaw**

The unprotected field is `is_admin_task`. The handler overrides `owner_id` but never enforces `is_admin_task = False`. A client can POST `{"title": "x", "is_admin_task": true}` and the value is stored exactly as sent, bypassing the server's intent.

**Task 2b: Explain the Mechanism**

1. Pydantic's job is type validation, not authorization. `is_admin_task: bool = False` tells Pydantic to accept any valid boolean. Whether a boolean value is *permitted* from a client is a design concern, not a validation concern. Pydantic correctly validates `true` as a valid `bool` and accepts it.
2. Mass assignment means a client populates a model field that the server never intended to expose for writing. The vulnerability originates in the model design: using one model for both input and output implicitly grants write access to every field the model declares.
3. Overriding every sensitive field in every handler is brittle. It requires every developer who writes a handler to know which fields are server-controlled — and to not forget any of them. Adding a new sensitive field to the model means auditing every handler that uses it. Separate input models create a single, explicit allowlist that is impossible to accidentally bypass.

**Task 2c: Instrument to Prove Diagnosis**

```python
def test_attacker_can_set_privileged_field():
    r = client.post("/v1/tasks", json={
        "title": "Normal-looking task",
        "is_admin_task": True,    # ← The exploit
    })
    assert r.status_code == 201
    assert r.json()["is_admin_task"] is True   # ← PASSES on broken code
```

**Task 2d: Fixed Code**

```python
from pydantic import BaseModel, Field
from typing import Optional

class TaskCreate(BaseModel):
    """The ONLY fields a client may provide when creating a task."""
    title: str = Field(..., min_length=1, max_length=200)
    description: Optional[str] = None
    priority: int = Field(default=1, ge=1, le=10)
    # Intentionally absent: owner_id, is_admin_task, status


class TaskResponse(BaseModel):
    """Complete task shape returned by every read endpoint."""
    id: int
    title: str
    description: Optional[str]
    priority: int
    status: str
    owner_id: int
    is_admin_task: bool   # Readable by client, never writable via create


@app.post("/v1/tasks", status_code=201, response_model=TaskResponse)
def create_task(task: TaskCreate):    # ← Only TaskCreate accepted as input
    global _next_id
    new = {
        "id": _next_id,
        "title": task.title,
        "description": task.description,
        "priority": task.priority,
        "status": "pending",           # ← Always server-enforced
        "owner_id": _current_user_id(),
        "is_admin_task": False,        # ← Always server-enforced; never from client
    }
    _next_id += 1
    TASKS.append(new)
    return new
```

Pydantic silently ignores extra fields not present in `TaskCreate`. The `response_model=TaskResponse` acts as a second-line defence, stripping any accidental extras before the response is sent.

**Task 2e: Verification Test**

```python
def test_client_cannot_set_privileged_field():
    r = client.post("/v1/tasks", json={
        "title": "Sneaky task",
        "is_admin_task": True,
    })
    assert r.status_code == 201
    assert r.json()["is_admin_task"] is False   # FAILS on broken, PASSES on fixed
```

---

## Solution 2.2: The Backwards Filter

**Task 2a: Identify the Flaw**

The bug is this line, which executes before any filter is applied:
```python
page = NOTES[offset : offset + limit]    # ← Pagination runs first
```

The violated invariant: `GET /notes?status=active&limit=5` implies a contract of "give me 5 active notes." The current code gives the client "up to 5 notes from a 5-note window, then filtered," which can yield fewer than the requested `limit` without the dataset being exhausted.

**Task 2b: Step-by-step trace:**

NOTES (first 5): `[1(active), 2(archived), 3(active), 4(archived), 5(active)]`

With `status=active, limit=5, offset=0`:
1. After pagination: `NOTES[0:5]` = 5 notes — IDs 1, 2, 3, 4, 5
2. After filter (active only): IDs 1, 3, 5 — only 3 items remain

The client received 3 items and must issue another request at `offset=5` to potentially find more — but that page may also be cut by the same pattern. The client has no way to know if 3 means "only 3 active notes exist" or "3 survived the window filter."

**Task 2c: Instrumented Output**

```
After pagination: 5 items — IDs: [1, 2, 3, 4, 5]
After filter:     3 items — IDs: [1, 3, 5]
```

3 items returned instead of 5. The filter discarded 2 items that were already inside the pagination window, and no mechanism refills those slots.

**Task 2d: Fixed Code**

```python
@app.get("/notes")
def list_notes_fixed(
    status: str | None = Query(default=None),
    limit: int = Query(default=5, ge=1),
    offset: int = Query(default=0, ge=0),
):
    # Step 1: Filter FIRST
    data = NOTES
    if status is not None:
        data = [n for n in data if n["status"] == status]

    total_matching = len(data)

    # Step 2: Then paginate
    page = data[offset : offset + limit]

    return {
        "items": page,
        "returned": len(page),
        "requested": limit,
        "total_matching": total_matching,
    }
```

**Task 2e: Verification Test**

```python
def test_filter_returns_full_page():
    r = client.get("/notes?status=active&limit=5&offset=0")
    data = r.json()
    assert data["returned"] == 5           # Exactly 5 active notes — FAILS on broken
    assert data["total_matching"] == 10    # 10 active notes in dataset
    assert all(n["status"] == "active" for n in data["items"])
```

---

# LEVEL 2.5: EXTEND EXISTING CODE

**Given:** A working cursor-paginated notes endpoint. The implementation is correct for the basic path. You will extend it in three stages of increasing difficulty. **The pre-written tests below must continue to pass after every extension.**

```python
# notes_service.py — Given implementation
from fastapi import FastAPI, Query, HTTPException, Request, Header, Depends
from fastapi.responses import JSONResponse, Response
from fastapi.testclient import TestClient
from pydantic import BaseModel
from typing import Optional
import base64, hashlib, json

app = FastAPI()

NOTES = [
    {
        "id": i,
        "title": f"Note {i}",
        "body": f"Body of note {i}",
        "status": "active" if i % 4 != 0 else "archived",
        "owner_id": 1 if i % 3 != 0 else 2,
    }
    for i in range(1, 16)    # 15 notes total
]
# owner_id=1: ids 1,2,4,5,7,8,10,11,13,14  (10 notes)
# owner_id=2: ids 3,6,9,12,15              (5 notes)
# archived:   ids 4, 8, 12                 (3 notes)


def encode_cursor(note_id: int) -> str:
    return base64.urlsafe_b64encode(f"id:{note_id}".encode()).decode()


def decode_cursor(cursor: str) -> int:
    try:
        raw = base64.urlsafe_b64decode(cursor.encode()).decode()
        return int(raw.split(":")[1])
    except Exception:
        raise HTTPException(status_code=400, detail="Invalid cursor format")


class PaginationMeta(BaseModel):
    has_more: bool
    next_cursor: Optional[str]


class NotesPage(BaseModel):
    items: list[dict]
    pagination: PaginationMeta


@app.get("/notes", response_model=NotesPage)
def list_notes(
    limit: int = Query(default=5, ge=1, le=10),
    cursor: Optional[str] = Query(default=None),
):
    data = NOTES    # No filters applied

    if cursor is not None:
        last_id = decode_cursor(cursor)
        start = next(
            (i + 1 for i, n in enumerate(data) if n["id"] == last_id),
            len(data),
        )
    else:
        start = 0

    page = data[start : start + limit]
    has_more = (start + limit) < len(data)
    next_cursor = encode_cursor(page[-1]["id"]) if page and has_more else None

    return NotesPage(
        items=page,
        pagination=PaginationMeta(has_more=has_more, next_cursor=next_cursor),
    )


@app.get("/notes/{note_id}")
def get_note(note_id: int):
    note = next((n for n in NOTES if n["id"] == note_id), None)
    if note is None:
        raise HTTPException(status_code=404, detail=f"Note {note_id} not found")
    return note


# ── PRE-WRITTEN TESTS — must pass after every extension ──────────────────
client = TestClient(app)

def test_first_page_returns_limit():
    r = client.get("/notes?limit=5")
    assert r.status_code == 200
    assert len(r.json()["items"]) == 5

def test_cursor_produces_no_duplicates():
    r1 = client.get("/notes?limit=5")
    cursor = r1.json()["pagination"]["next_cursor"]
    r2 = client.get(f"/notes?limit=5&cursor={cursor}")
    ids1 = {n["id"] for n in r1.json()["items"]}
    ids2 = {n["id"] for n in r2.json()["items"]}
    assert len(ids1 & ids2) == 0

def test_last_page_has_no_cursor():
    r = client.get("/notes?limit=15")
    assert r.json()["pagination"]["has_more"] is False
    assert r.json()["pagination"]["next_cursor"] is None
```

---

**Extension 1 — Add `?status=` filter (Medium)**

Add an optional `status` query parameter. When provided, only notes matching that status appear on each page. The cursor must continue to work correctly *within the filtered result set* — consecutive pages of active notes must not contain archived notes and must not skip any active notes.

New tests that must pass after Extension 1:
```python
def test_status_filter_only_returns_matching():
    r = client.get("/notes?status=active&limit=5")
    assert all(n["status"] == "active" for n in r.json()["items"])

def test_cursor_works_within_filtered_set():
    r1 = client.get("/notes?status=active&limit=5")
    cursor = r1.json()["pagination"]["next_cursor"]
    if cursor:
        r2 = client.get(f"/notes?status=active&limit=5&cursor={cursor}")
        ids1 = {n["id"] for n in r1.json()["items"]}
        ids2 = {n["id"] for n in r2.json()["items"]}
        assert len(ids1 & ids2) == 0
        assert all(n["status"] == "active" for n in r2.json()["items"])
```

The three existing tests (without a status filter) must still pass.

---

**Extension 2 — Add ownership via `X-User-Id` header (Hard)**

Enforce ownership. Use an `X-User-Id` integer header as a stub for the current user. Extend both `/notes` and `/notes/{note_id}`.

Rules:
1. Missing or non-integer `X-User-Id` → `401 Unauthorized`
2. `/notes` returns only notes where `owner_id` matches the header value
3. `/notes/{note_id}` returns `404` — not `403` — when the note belongs to a different user

New tests that must pass after Extension 2:
```python
def test_missing_header_returns_401():
    r = client.get("/notes")    # No header
    assert r.status_code == 401

def test_user_only_sees_own_notes():
    r = client.get("/notes?limit=15", headers={"X-User-Id": "1"})
    owner_ids = {n["owner_id"] for n in r.json()["items"]}
    assert owner_ids == {1}

def test_accessing_other_users_note_returns_404_not_403():
    # Note 3 belongs to owner_id=2. User 1 requests it.
    r = client.get("/notes/3", headers={"X-User-Id": "1"})
    assert r.status_code == 404
    assert r.status_code != 403    # 403 would confirm the note exists
```

The existing three tests must pass when called with `headers={"X-User-Id": "1"}`.

---

**Extension 3 — Add ETag to `GET /notes/{note_id}` (Hard)**

Implement conditional GET for single-note retrieval:
1. Every `200 OK` response from `GET /notes/{note_id}` must include an `ETag` header
2. A request with `If-None-Match` matching the current ETag must return `304 Not Modified` with an empty body
3. After a note's data changes, its ETag must change — a stale `If-None-Match` must receive a `200` with the updated data and a new ETag

New tests that must pass after Extension 3:
```python
def test_200_includes_etag():
    r = client.get("/notes/1", headers={"X-User-Id": "1"})
    assert r.status_code == 200
    assert "ETag" in r.headers

def test_304_on_matching_etag():
    r1 = client.get("/notes/1", headers={"X-User-Id": "1"})
    etag = r1.headers["ETag"]
    r2 = client.get("/notes/1", headers={"X-User-Id": "1", "If-None-Match": etag})
    assert r2.status_code == 304
    assert r2.content == b""    # Strictly empty body

def test_stale_etag_returns_200_with_new_etag():
    r1 = client.get("/notes/1", headers={"X-User-Id": "1"})
    old_etag = r1.headers["ETag"]

    # Simulate a data change
    NOTES[0]["title"] = "Title changed after first fetch"

    r2 = client.get("/notes/1", headers={"X-User-Id": "1", "If-None-Match": old_etag})
    assert r2.status_code == 200               # Data changed → full response
    assert r2.headers["ETag"] != old_etag      # Hash must reflect new data
```

---

# LEVEL 2.5 SOLUTIONS

## Extension 1 Solution

```python
@app.get("/notes", response_model=NotesPage)
def list_notes(
    limit: int = Query(default=5, ge=1, le=10),
    cursor: Optional[str] = Query(default=None),
    status: Optional[str] = Query(default=None),    # ← New
):
    # ① Apply status filter BEFORE cursor resolution
    data = NOTES
    if status is not None:
        data = [n for n in data if n["status"] == status]

    # ② Resolve cursor within the filtered data
    if cursor is not None:
        last_id = decode_cursor(cursor)
        start = next(
            (i + 1 for i, n in enumerate(data) if n["id"] == last_id),
            len(data),
        )
    else:
        start = 0

    # ③ Paginate the filtered, cursor-positioned data
    page = data[start : start + limit]
    has_more = (start + limit) < len(data)
    next_cursor = encode_cursor(page[-1]["id"]) if page and has_more else None

    return NotesPage(
        items=page,
        pagination=PaginationMeta(has_more=has_more, next_cursor=next_cursor),
    )
```

**Key Insight:** If you resolve the cursor in the unfiltered list and apply the filter afterward, the cursor position in the full list no longer corresponds to positions in the filtered subset. You will skip active notes whose IDs happen to fall in the gap after an archived note's cursor position. Filter first, then find the cursor within the filtered result — the cursor is an address in the *post-filter* sequence, not in the raw database order.

---

## Extension 2 Solution

```python
def get_current_user_id(x_user_id: Optional[str] = Header(default=None)) -> int:
    if x_user_id is None:
        raise HTTPException(status_code=401, detail="X-User-Id header required")
    try:
        return int(x_user_id)
    except ValueError:
        raise HTTPException(status_code=401, detail="X-User-Id must be an integer")


@app.get("/notes", response_model=NotesPage)
def list_notes(
    limit: int = Query(default=5, ge=1, le=10),
    cursor: Optional[str] = Query(default=None),
    status: Optional[str] = Query(default=None),
    user_id: int = Depends(get_current_user_id),    # ← Auth
):
    # ① OWNERSHIP — always first, always unconditional
    data = [n for n in NOTES if n["owner_id"] == user_id]

    # ② Client-requested filters on the already-narrowed set
    if status is not None:
        data = [n for n in data if n["status"] == status]

    # ③ Cursor resolution
    if cursor is not None:
        last_id = decode_cursor(cursor)
        start = next(
            (i + 1 for i, n in enumerate(data) if n["id"] == last_id),
            len(data),
        )
    else:
        start = 0

    # ④ Paginate
    page = data[start : start + limit]
    has_more = (start + limit) < len(data)
    next_cursor = encode_cursor(page[-1]["id"]) if page and has_more else None

    return NotesPage(
        items=page,
        pagination=PaginationMeta(has_more=has_more, next_cursor=next_cursor),
    )


@app.get("/notes/{note_id}")
def get_note(
    note_id: int,
    user_id: int = Depends(get_current_user_id),
):
    note = next((n for n in NOTES if n["id"] == note_id), None)
    if note is None or note["owner_id"] != user_id:
        # Same response for "doesn't exist" and "not your note"
        raise HTTPException(status_code=404, detail=f"Note {note_id} not found")
    return note
```

**Key Insight:** The ownership filter is step ①. It is not a parameter, not togglable, not one filter among many. Placing it second — after any client-requested filter — means step ① operates on data already narrowed by the client's filter. If the client can control what enters step ①, they control whose data gets narrowed. Ownership must reduce the universe before any client-controlled operation touches it.

---

## Extension 3 Solution

```python
def generate_etag(data: dict) -> str:
    # sort_keys=True ensures the same data always produces the same hash
    # regardless of Python dictionary insertion order
    serialized = json.dumps(data, sort_keys=True, default=str)
    digest = hashlib.sha256(serialized.encode()).hexdigest()[:16]
    return f'"{digest}"'    # ETags must always be quoted


@app.get("/notes/{note_id}")
def get_note(
    note_id: int,
    request: Request,
    user_id: int = Depends(get_current_user_id),
):
    note = next((n for n in NOTES if n["id"] == note_id), None)
    if note is None or note["owner_id"] != user_id:
        raise HTTPException(status_code=404, detail=f"Note {note_id} not found")

    etag = generate_etag(note)

    if request.headers.get("If-None-Match") == etag:
        return Response(status_code=304)    # No body. No Content-Type.

    return JSONResponse(
        content=note,
        headers={
            "ETag": etag,
            "Cache-Control": "private, max-age=0, must-revalidate",
        },
    )
```

**Key Insight:** The ETag must appear on the **200 response**. A conditional request chain has two phases: the server sends ETag on 200, the client stores it, and on the next request the client sends `If-None-Match`. If phase one never sends the ETag, phase two can never begin — the client has nothing to send. Every 304 path in the world is useless without a prior 200 that included the ETag header.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## The Scenario

You are building the backend for a **personal bookmark manager**. Users save URLs with a title and an optional note. Multiple users share the same API instance. The product requirements are:

- Users must never see each other's bookmarks
- Pages of bookmarks must not duplicate or skip items even if new bookmarks are created between page requests
- A user must be able to clear a bookmark's `note` field by sending `{"note": null}`
- Repeated identical GET requests for the same bookmark must return `304` when nothing has changed
- Error responses must follow RFC 7807 structure

Each requirement corresponds to at least one failing test if you use a naive approach.

```python
# bookmark_api.py — Starter scaffold. Complete the implementation.
from fastapi import FastAPI, Query, HTTPException, Header, Request, Depends
from fastapi.responses import JSONResponse, Response
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
from typing import Optional
import base64, hashlib, json

app = FastAPI()

# ── Seed data ─────────────────────────────────────────────────────────────
BOOKMARKS = [
    {
        "id": i,
        "url": f"https://example.com/resource/{i}",
        "title": f"Bookmark {i}",
        "note": f"Note for bookmark {i}" if i % 3 != 0 else None,
        "owner_id": 1 if i % 2 != 0 else 2,
    }
    for i in range(1, 13)   # 12 bookmarks
]
# User 1 owns odd  IDs: 1, 3, 5, 7, 9, 11   (6 bookmarks)
# User 2 owns even IDs: 2, 4, 6, 8, 10, 12  (6 bookmarks)
# note=None for IDs divisible by 3: 3, 6, 9, 12


# ── Models ────────────────────────────────────────────────────────────────
# Define your models here.
# You need at minimum: BookmarkCreate, BookmarkUpdate, BookmarkResponse
# and a pagination envelope.


# ── Auth dependency ───────────────────────────────────────────────────────
# Implement get_current_user_id(x_user_id: Optional[str] = Header(...)) -> int
# Raise 401 if missing or not a valid integer.


# ── Exception handler ─────────────────────────────────────────────────────
# Register an @app.exception_handler(HTTPException) that returns RFC 7807 shape:
# { "type": str, "title": str, "status": int, "detail": str }


# ── Endpoints to implement ────────────────────────────────────────────────
# POST   /v1/bookmarks           Create a bookmark
# GET    /v1/bookmarks           List (cursor-paginated, ownership-filtered)
# GET    /v1/bookmarks/{id}      Single bookmark (ETag-cached, ownership-aware)
# PATCH  /v1/bookmarks/{id}      Partial update (exclude_unset, null-clearing)
```

---

**Task 3a: Pass the basic tests**

Implement the endpoints such that these tests pass:

```python
client = TestClient(app)

def test_create_sets_owner_from_auth_not_client():
    r = client.post(
        "/v1/bookmarks",
        json={"url": "https://test.com", "title": "Test Bookmark"},
        headers={"X-User-Id": "1"},
    )
    assert r.status_code == 201
    assert r.json()["owner_id"] == 1    # Must come from header, never from body

def test_unauthenticated_request_returns_401():
    r = client.get("/v1/bookmarks")    # No X-User-Id header
    assert r.status_code == 401

def test_list_returns_only_own_bookmarks():
    r = client.get("/v1/bookmarks?limit=10", headers={"X-User-Id": "1"})
    assert r.status_code == 200
    owner_ids = {b["owner_id"] for b in r.json()["items"]}
    assert owner_ids == {1}

def test_other_users_bookmark_returns_404():
    # Bookmark 2 belongs to user 2. User 1 requests it.
    r = client.get("/v1/bookmarks/2", headers={"X-User-Id": "1"})
    assert r.status_code == 404

def test_error_response_has_rfc7807_shape():
    r = client.get("/v1/bookmarks/99999", headers={"X-User-Id": "1"})
    assert r.status_code == 404
    body = r.json()
    for key in ("type", "title", "status", "detail"):
        assert key in body, f"RFC 7807 key missing: {key!r}"
    assert body["status"] == 404
```

---

**Task 3b: Pass the pagination and PATCH tests**

```python
def test_cursor_pagination_no_duplicates():
    r1 = client.get("/v1/bookmarks?limit=3", headers={"X-User-Id": "1"})
    cursor = r1.json()["pagination"]["next_cursor"]
    r2 = client.get(f"/v1/bookmarks?limit=3&cursor={cursor}",
                    headers={"X-User-Id": "1"})
    ids1 = {b["id"] for b in r1.json()["items"]}
    ids2 = {b["id"] for b in r2.json()["items"]}
    assert len(ids1 & ids2) == 0    # Zero overlap between consecutive pages

def test_patch_preserves_unsent_fields():
    # Bookmark 1 belongs to user 1 and has a note (1 % 3 != 0)
    r = client.patch(
        "/v1/bookmarks/1",
        json={"title": "Updated title"},
        headers={"X-User-Id": "1"},
    )
    assert r.status_code == 200
    assert r.json()["title"] == "Updated title"
    assert r.json()["note"] is not None    # Note must be preserved

def test_patch_can_clear_nullable_field():
    # Bookmark 1 has a note. Client explicitly sends {"note": null}.
    r = client.patch(
        "/v1/bookmarks/1",
        json={"note": None},
        headers={"X-User-Id": "1"},
    )
    assert r.status_code == 200
    assert r.json()["note"] is None    # Note must be cleared, not preserved
```

---

**Task 3c: Pass the ETag tests**

```python
def test_200_response_includes_etag():
    r = client.get("/v1/bookmarks/1", headers={"X-User-Id": "1"})
    assert r.status_code == 200
    assert "ETag" in r.headers

def test_304_on_matching_etag():
    r1 = client.get("/v1/bookmarks/1", headers={"X-User-Id": "1"})
    etag = r1.headers["ETag"]

    r2 = client.get(
        "/v1/bookmarks/1",
        headers={"X-User-Id": "1", "If-None-Match": etag},
    )
    assert r2.status_code == 304
    assert r2.content == b""

def test_200_after_patch_invalidates_etag():
    r1 = client.get("/v1/bookmarks/1", headers={"X-User-Id": "1"})
    old_etag = r1.headers["ETag"]

    client.patch(
        "/v1/bookmarks/1",
        json={"title": "Title changed to invalidate cache"},
        headers={"X-User-Id": "1"},
    )

    r2 = client.get(
        "/v1/bookmarks/1",
        headers={"X-User-Id": "1", "If-None-Match": old_etag},
    )
    assert r2.status_code == 200              # Data changed — full response
    assert r2.headers["ETag"] != old_etag    # New ETag reflects new data
```

---

# LEVEL 3 SOLUTIONS

```python
# ── Models ────────────────────────────────────────────────────────────────

class BookmarkCreate(BaseModel):
    url: str = Field(..., min_length=1)
    title: str = Field(..., min_length=1, max_length=200)
    note: Optional[str] = None
    # Intentionally absent: owner_id (server-set), id (auto-generated)


class BookmarkUpdate(BaseModel):
    title: Optional[str] = Field(default=None, min_length=1, max_length=200)
    note: Optional[str] = None    # Nullable — client may clear it with {"note": null}
    # url intentionally absent — URLs are immutable after creation


class BookmarkResponse(BaseModel):
    id: int
    url: str
    title: str
    note: Optional[str]
    owner_id: int


class PaginationMeta(BaseModel):
    has_more: bool
    next_cursor: Optional[str]


class BookmarksPage(BaseModel):
    items: list[BookmarkResponse]
    pagination: PaginationMeta


class ProblemDetail(BaseModel):
    type: str
    title: str
    status: int
    detail: str


# ── Helpers ───────────────────────────────────────────────────────────────

def encode_cursor(bookmark_id: int) -> str:
    return base64.urlsafe_b64encode(f"id:{bookmark_id}".encode()).decode()


def decode_cursor(cursor: str) -> int:
    try:
        raw = base64.urlsafe_b64decode(cursor.encode()).decode()
        return int(raw.split(":")[1])
    except Exception:
        raise HTTPException(status_code=400, detail="Invalid cursor format")


def generate_etag(data: dict) -> str:
    serialized = json.dumps(data, sort_keys=True, default=str)
    return f'"{hashlib.sha256(serialized.encode()).hexdigest()[:16]}"'


_STATUS_TITLES = {
    400: "Bad Request", 401: "Not Authenticated", 403: "Forbidden",
    404: "Not Found", 422: "Validation Failed", 500: "Internal Server Error",
}


# ── Auth dependency ───────────────────────────────────────────────────────

def get_current_user_id(x_user_id: Optional[str] = Header(default=None)) -> int:
    if x_user_id is None:
        raise HTTPException(status_code=401, detail="X-User-Id header required")
    try:
        return int(x_user_id)
    except ValueError:
        raise HTTPException(status_code=401, detail="X-User-Id must be an integer")


# ── RFC 7807 exception handler ────────────────────────────────────────────

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    problem = ProblemDetail(
        type=f"https://api.example.com/problems/{exc.status_code}",
        title=_STATUS_TITLES.get(exc.status_code, f"HTTP {exc.status_code}"),
        status=exc.status_code,
        detail=str(exc.detail),
    )
    return JSONResponse(
        status_code=exc.status_code,
        content=problem.model_dump(),
        headers={"Content-Type": "application/problem+json"},
    )


# ── Endpoints ─────────────────────────────────────────────────────────────

_next_id = 13    # Continues from seed data

@app.post("/v1/bookmarks", status_code=201, response_model=BookmarkResponse)
def create_bookmark(
    body: BookmarkCreate,
    user_id: int = Depends(get_current_user_id),
):
    global _next_id
    new = {
        "id": _next_id,
        "url": body.url,
        "title": body.title,
        "note": body.note,
        "owner_id": user_id,    # Always from auth dependency, never from body
    }
    _next_id += 1
    BOOKMARKS.append(new)
    return new


@app.get("/v1/bookmarks", response_model=BookmarksPage)
def list_bookmarks(
    limit: int = Query(default=5, ge=1, le=20),
    cursor: Optional[str] = Query(default=None),
    user_id: int = Depends(get_current_user_id),
):
    # ① OWNERSHIP — always step one
    data = [b for b in BOOKMARKS if b["owner_id"] == user_id]

    # ② Cursor resolution within the owner's bookmarks
    if cursor is not None:
        last_id = decode_cursor(cursor)
        start = next(
            (i + 1 for i, b in enumerate(data) if b["id"] == last_id),
            len(data),
        )
    else:
        start = 0

    # ③ Paginate
    page = data[start : start + limit]
    has_more = (start + limit) < len(data)
    next_cursor = encode_cursor(page[-1]["id"]) if page and has_more else None

    return BookmarksPage(
        items=page,
        pagination=PaginationMeta(has_more=has_more, next_cursor=next_cursor),
    )


@app.get("/v1/bookmarks/{bookmark_id}", response_model=BookmarkResponse)
def get_bookmark(
    bookmark_id: int,
    request: Request,
    user_id: int = Depends(get_current_user_id),
):
    bookmark = next((b for b in BOOKMARKS if b["id"] == bookmark_id), None)
    if bookmark is None or bookmark["owner_id"] != user_id:
        raise HTTPException(status_code=404, detail=f"Bookmark {bookmark_id} not found")

    etag = generate_etag(bookmark)

    if request.headers.get("If-None-Match") == etag:
        return Response(status_code=304)

    return JSONResponse(
        content=bookmark,
        headers={
            "ETag": etag,
            "Cache-Control": "private, max-age=0, must-revalidate",
        },
    )


@app.patch("/v1/bookmarks/{bookmark_id}", response_model=BookmarkResponse)
def update_bookmark(
    bookmark_id: int,
    update: BookmarkUpdate,
    user_id: int = Depends(get_current_user_id),
):
    bookmark = next((b for b in BOOKMARKS if b["id"] == bookmark_id), None)
    if bookmark is None or bookmark["owner_id"] != user_id:
        raise HTTPException(status_code=404, detail=f"Bookmark {bookmark_id} not found")

    # Only apply fields the client actually included in the JSON body
    update_data = update.model_dump(exclude_unset=True)
    if not update_data:
        raise HTTPException(status_code=422, detail="PATCH body must contain at least one field")

    bookmark.update(update_data)
    return bookmark
```

**Why each decision is non-negotiable for the tests:**

- `BookmarkCreate` omits `owner_id` → `test_create_sets_owner_from_auth_not_client` passes because the handler always takes it from the dependency
- `model_dump(exclude_unset=True)` in PATCH → `test_patch_can_clear_nullable_field` passes because `{"note": null}` is represented as `{"note": None}` in the dump, while `test_patch_preserves_unsent_fields` passes because `title` is absent from the dump entirely
- `404` for `note.owner_id != user_id` → `test_other_users_bookmark_returns_404` passes and `test_error_response_has_rfc7807_shape` passes because the exception handler converts every `HTTPException` to the RFC 7807 shape
- `ETag` sent on every `200` → `test_304_on_matching_etag` can only work if the prior `200` included the header that the client stores and resends

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## When Cursor Pagination Fails

**Setup:** After reading this lecture, a junior engineer proposes converting every paginated endpoint in your platform to cursor-based pagination, including the admin user management dashboard, which currently shows