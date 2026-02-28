# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "Pydantic is a symmetric type-converter. If I annotate a field as `str`, it will accept and convert anything into a string. All operations on a model — construction, copy, inheritance — pass through the same validation pipeline. Once I define a model, I can reuse it everywhere for the same entity, since all the fields are already there."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---|---|
| 1 | Pydantic coerces bi-directionally: `"42"` → `int 42` AND `int 42` → `"42"` | Pydantic v2 coerces `str → int` but **NOT** `int → str`. The asymmetry is a deliberate design decision. |
| 2 | A `@field_validator` that calls `v.strip()` or `v.lower()` modifies the field in-place | Strings are immutable. `.strip()` returns a **new** string. Without `return`, the function implicitly returns `None`, and that `None` becomes the field value. |
| 3 | `model_copy(update={"balance": -500})` triggers validation and raises `ValidationError` for `Field(ge=0)` | `model_copy` bypasses **all** validation by design. Invalid data enters the model silently. |
| 4 | `Union[ModelA, ModelB]` picks the "best fit" — the model that most completely matches the input data | Union tries models left-to-right. The **first match wins**, regardless of whether a later model fits better. Extra fields from the input are silently dropped. |
| 5 | A `list["Model"]` forward reference automatically validates nested instances once the class is defined | Without `Model.model_rebuild()`, nested items remain as plain `dict` objects — no type checking, no method access, no nested validation. No error is raised. |
| 6 | One Pydantic model can safely cover registration input, API response, and partial update for the same entity | The three operations require different required/optional/excluded fields. One model leaks secrets, generates misleading OpenAPI docs, and breaks `PATCH` semantics. |
| 7 | `model_validate_json()` and `model_validate()` are interchangeable; both handle dicts and JSON strings | `model_validate_json()` requires `str` or `bytes`. Passing a dict raises a confusing `ValidationError` about model type. |

**Prediction Collision Map:**

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "str coerces anything" | `User(name=42)` creates user with `name="42"` | `ValidationError: Input should be a valid string` | v2 rejects `int → str`; it almost always signals a bug, not a format need |
| "validator runs my logic" | `v.strip()` in validator cleans the field | Field becomes `None` | `.strip()` returns a new string; without `return`, function returns `None` |
| "copy re-validates" | `account.model_copy(update={"balance": -500})` raises `ValidationError` | No error; `balance = -500` | `model_copy` is for trusted internal copies; re-validating would be overhead on already-valid data |
| "Union finds best fit" | Ambiguous data is routed to the most specific matching model | First matching model wins; unmatched fields silently vanish | Union is sequential, not score-based |
| "forward ref just works" | Nested `"Comment"` dicts become `Comment` instances | Plain dicts; `.attribute` access raises `AttributeError` | Schema is only built after `model_rebuild()` is called |

---

## The Main Purpose (Essence)

**Core Insight:** Pydantic is a **disciplined contract enforcer**, not a permissive autocorrector. The contract has two halves: data *entering* the model is guaranteed correct by the time it reaches your code; data *leaving* the model is guaranteed safe — it contains only what you explicitly permitted. Both guarantees require understanding the exact pipeline data travels through and, critically, which operations are *outside* that pipeline.

**The Key Distinctions:**

- **Coercion vs. Correction**: Pydantic coerces within reasonable type families (`"25"` → `int 25`). It refuses conversions that almost always indicate a bug (`int 42` → `"42"`). When in doubt, it rejects rather than silently corrects.
- **Validation at construction vs. model_copy**: `__init__` and `model_validate` validate. `model_copy` does not. This is by design: copy is for trusted internal operations; external input must always pass through a validating constructor.
- **One model vs. three models**: The Create/Read/Update split is not code duplication — it is precise modeling of three fundamentally different data contracts. A registration input model carries a plaintext password; a response model must never include a password hash; an update model has every field optional to support `PATCH` semantics. Collapsing these into one model simultaneously leaks secrets, pollutes API documentation, and corrupts partial updates.

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Asymmetric Gate

```python
from pydantic import BaseModel

class ServerConfig(BaseModel):
    host: str
    port: int
    max_workers: int
    debug: bool

# Scenario A — numeric strings for int and bool fields
try:
    config_a = ServerConfig(
        host="localhost",
        port="8080",        # str → int field
        max_workers="4",    # str → int field
        debug="true"        # str → bool field
    )
    print(f"A: port={config_a.port!r}, type={type(config_a.port).__name__}")
    print(f"A: debug={config_a.debug!r}, type={type(config_a.debug).__name__}")
except Exception as e:
    print(f"A ERROR: {e}")

# Scenario B — integer for a str field
try:
    config_b = ServerConfig(
        host=127001,        # int → str field
        port=8080,
        max_workers=4,
        debug=False
    )
    print(f"B: host={config_b.host!r}")
except Exception as e:
    print(f"B ERROR: {e}")

# Scenario C — "1" for a bool field, and int 0 for a bool field
try:
    config_c = ServerConfig(host="localhost", port=8080, max_workers=4, debug="1")
    config_d = ServerConfig(host="localhost", port=8080, max_workers=4, debug=0)
    print(f"C: debug={config_c.debug!r}")
    print(f"D: debug={config_d.debug!r}")
except Exception as e:
    print(f"C/D ERROR: {e}")
```

**Questions:**

1. **Predict the exact output.** Which scenarios succeed and which raise errors? For Scenario A, what are the Python types and values of `config_a.port` and `config_a.debug`? Write each print line exactly as it would appear.

2. **Explain the asymmetry.** Scenario A successfully converts `"8080"` (a string) to the integer `8080`. Scenario B fails to convert `127001` (an integer) to a string. Pydantic's coercion is intentionally one-directional here. Why does Pydantic v2 allow `str → int` but refuse `int → str`? What principle drives this decision?

3. **Explain the special cases.** In Scenario C, what does `debug="1"` produce — the string `"1"`, the boolean `True`, or a `ValidationError`? What about `debug=0` in Scenario D? Why does Pydantic handle these specific values for `bool` fields?

---

## Exercise 1.2: The Invisible Null

```python
from pydantic import BaseModel, Field, field_validator

class Article(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    slug: str = Field(min_length=1, max_length=200)
    author: str

    @field_validator("title", "slug")
    @classmethod
    def strip_and_lowercase(cls, v: str) -> str:
        v.strip()
        v.lower()

    @field_validator("author")
    @classmethod
    def normalize_author(cls, v: str) -> str:
        return v.strip().title()

article = Article(
    title="  MY FIRST POST  ",
    slug="  MY-FIRST-POST  ",
    author="  alice smith  "
)

print(f"title:      {article.title!r}")
print(f"slug:       {article.slug!r}")
print(f"author:     {article.author!r}")
print(f"title None: {article.title is None}")
```

**Questions:**

1. **Predict the exact output.** What does each line print? Pay particular attention to `title` and `slug`. Are they the original padded strings, the stripped-and-lowercased strings, or something else entirely?

2. **Explain why.** The `strip_and_lowercase` validator clearly executes `v.strip()` and `v.lower()`. The validator runs — so why does `title` end up as `None` instead of `"my first post"`? Where exactly does the data go wrong?

3. **The contrast.** The `normalize_author` validator works correctly, producing `"Alice Smith"`. Both validators have the same function signature and purpose. Identify the single structural difference that makes `normalize_author` correct and `strip_and_lowercase` silently destructive.

4. **Mental trace.** Walk through each line of `strip_and_lowercase` when called with `v = "  MY FIRST POST  "`. For each line, write: what value is computed, where it goes, and what the local variable `v` holds afterward. Finish by stating what the function returns.

---

## Exercise 1.3: The Trusting Copy

```python
from pydantic import BaseModel, Field

class BankAccount(BaseModel):
    account_id: str
    owner: str = Field(min_length=1, max_length=50)
    balance: float = Field(ge=0, description="Cannot be negative")
    is_frozen: bool = False

def apply_withdrawal(account: BankAccount, amount: float) -> BankAccount:
    """Returns a new account state after a withdrawal."""
    return account.model_copy(update={"balance": account.balance - amount})

def clear_owner(account: BankAccount) -> BankAccount:
    """Clears owner name (used in an account-anonymization routine)."""
    return account.model_copy(update={"owner": "", "is_frozen": True})

account = BankAccount(account_id="ACC-001", owner="Alice", balance=100.0)

# Scenario A: Overdraft via model_copy
try:
    overdrawn = apply_withdrawal(account, 500.0)
    print(f"A: balance = {overdrawn.balance}")
except Exception as e:
    print(f"A ERROR: {type(e).__name__}")

# Scenario B: Empty owner via model_copy
try:
    cleared = clear_owner(account)
    print(f"B: owner = {cleared.owner!r}, frozen = {cleared.is_frozen}")
except Exception as e:
    print(f"B ERROR: {type(e).__name__}")

# Scenario C: Direct construction with the same invalid data
try:
    invalid = BankAccount(account_id="ACC-002", owner="", balance=-50.0)
    print(f"C: created successfully")
except Exception as e:
    print(f"C ERROR: {type(e).__name__} raised")
```

**Questions:**

1. **Predict the exact output** for all three scenarios. Which of A, B, C raise exceptions, and which succeed?

2. **Explain the inconsistency.** Scenario C correctly rejects the same invalid data that Scenarios A and B accept without error. Why does `model_copy` behave differently from constructing a new instance? Is this a Pydantic bug?

3. **The security implication.** If `apply_withdrawal` is called from a FastAPI endpoint that accepts a user-submitted `amount`, what specific attack does this code enable? Describe the exact payload an attacker would send.

4. **Safe fix.** Rewrite `apply_withdrawal` so that the `Field(ge=0)` constraint on `balance` is enforced for the new value. You must not modify the `BankAccount` class definition. You may not use `model_copy`.

---

## Exercise 1.4: The First-Match Trap

```python
from typing import Union
from pydantic import BaseModel

class FreeShipping(BaseModel):
    method: str = "free"
    estimated_days: int = 7

class PaidShipping(BaseModel):
    method: str = "paid"
    cost: float
    estimated_days: int

class Order(BaseModel):
    item_name: str
    shipping: Union[FreeShipping, PaidShipping]

# Scenario A: Explicitly paid shipping data
order_a = Order(
    item_name="Widget",
    shipping={"method": "paid", "cost": 9.99, "estimated_days": 3}
)

# Scenario B: Explicitly free shipping data
order_b = Order(
    item_name="Gadget",
    shipping={"method": "free", "estimated_days": 7}
)

# Scenario C: Paid-looking data, no "method" key
order_c = Order(
    item_name="Doohickey",
    shipping={"cost": 14.99, "estimated_days": 2}
)

print(f"A type:   {type(order_a.shipping).__name__}")
print(f"A cost:   {getattr(order_a.shipping, 'cost', 'FIELD MISSING')}")
print(f"A method: {order_a.shipping.method!r}")
print()
print(f"B type:   {type(order_b.shipping).__name__}")
print()
print(f"C type:   {type(order_c.shipping).__name__}")
print(f"C cost:   {getattr(order_c.shipping, 'cost', 'FIELD MISSING')}")
print(f"C method: {order_c.shipping.method!r}")
```

**Questions:**

1. **Predict the exact output.** Think carefully about each scenario before answering. Scenario A is the most counterintuitive — trace it first.

2. **Trace Scenario A's Union resolution step by step.** The input is `{"method": "paid", "cost": 9.99, "estimated_days": 3}`. Walk through exactly what Pydantic does: which model is tried first, what happens to each field in the input, and why the result is what it is.

3. **What happened to the `cost` in Scenario A?** A developer calls `order_a.shipping.cost` expecting `9.99`. What do they get and why?

4. **The fix.** The root problem is that `FreeShipping` has no *required* fields — it will accept any dict. Describe the minimal redesign of `FreeShipping`, `PaidShipping`, and `Order` that makes Pydantic select the correct model deterministically and reject an unknown shipping method with a clear, specific error message. You must not remove any fields or use `model_config = ConfigDict(extra="forbid")` as the solution.

---

# Level 1 Solutions

## Solution 1.1: The Asymmetric Gate

**1. Exact output:**

```
A: port=8080, type=int
A: debug=True, type=bool
B ERROR: 1 validation error for ServerConfig
host
  Input should be a valid string [type=string_type, input_value=127001, input_type=int]
C: debug=True
D: debug=False
```

Scenario A fully succeeds. `"8080"` coerces to `int 8080`. `"true"` coerces to `bool True`. Scenario B raises a `ValidationError` — `int 127001` will not coerce to `str`. Scenarios C and D both succeed.

**2. The asymmetry explained:**

Pydantic v2 permits `str → int` because this conversion is unambiguous and common in real systems: web forms, HTTP query parameters, and some JSON sources encode numbers as strings. The intent is clear — the client means the number `8080`, not the string `"8080"`.

Pydantic v2 refuses `int → str` because it is almost always a **programming mistake**, not a format-conversion need. If your code passes `127001` to a field annotated as `str`, you likely have a mismatched type in your own code — not a client formatting issue. Silently converting `127001` to `"127001"` would hide this bug from you. The principle: *coerce what is unambiguous and common; reject what is almost certainly wrong.*

**3. The bool special cases:**

- `debug="1"` → `True`. Pydantic v2's bool validator (in lax mode) accepts the values `1`, `0`, `"1"`, `"0"`, `"true"`, `"false"`, `"on"`, `"off"`, `"yes"`, `"no"`. The string `"1"` is in the approved truthy set, so it coerces to `True`. This mirrors the conventions used in environment variables and config files.
- `debug=0` → `False`. The integer `0` is in the approved falsy set. The integer `1` would produce `True`. This is lax coercion for booleans specifically — the type family between `int` and `bool` is close enough in Python that Pydantic permits it.

---

## Solution 1.2: The Invisible Null

**1. Exact output:**

```
title:      None
slug:       None
author:     'Alice Smith'
title None: True
```

**2. Why title is `None`:**

Python strings are immutable objects. `v.strip()` does not modify `v` — it creates and returns a **brand new string**. The `strip_and_lowercase` validator calls `v.strip()` and `v.lower()`, but both return values are immediately discarded. No variable captures them. The function body then ends without a `return` statement.

In Python, a function without an explicit `return` returns `None`. Pydantic's `@field_validator` takes the function's return value and uses it as the new field value. So Pydantic sets `article.title = None`.

Crucially, **no error is raised**. The field passed type validation as a `str` before the validator ran; the validator's return value is accepted without a re-check. `None` silently replaces the input string.

**3. The structural difference:**

```python
# BROKEN — result of every expression is discarded
def strip_and_lowercase(cls, v: str) -> str:
    v.strip()   # ← New string created, immediately thrown away
    v.lower()   # ← New string created, immediately thrown away
    # Implicit: return None

# CORRECT — result is captured, transformed, and returned
def normalize_author(cls, v: str) -> str:
    return v.strip().title()  # ← Method chain returned directly
```

The single structural difference: `normalize_author` has a `return` statement. The type annotation `-> str` is documentation only — Pydantic v2 does not enforce that the validator actually returns a `str`.

**4. Mental trace:**

```
Call: strip_and_lowercase(cls, v="  MY FIRST POST  ")

Line 1: v.strip()
  → Computes "MY FIRST POST" (new string object)
  → Result is discarded; v is still "  MY FIRST POST  "

Line 2: v.lower()
  → Computes "  my first post  " (new string object)
  → Result is discarded; v is still "  MY FIRST POST  "

End of function body
  → No return statement
  → Python returns None implicitly

Pydantic receives: None
Pydantic sets: article.title = None
```

---

## Solution 1.3: The Trusting Copy

**1. Exact output:**

```
A: balance = -400.0
B: owner = '', frozen = True
C ERROR: ValidationError raised
```

**2. The design decision:**

This is intentional behavior, not a Pydantic bug. `model_copy` is designed for **trusted internal transformations** — moving data between pipeline stages where you, the programmer, know the values are valid and re-validating would be redundant overhead. Creating an instance from *external* input always validates; copying an *already-validated* instance for internal use does not.

The asymmetry is correct as a default: external data is untrusted (validate it), internal copies are trusted (skip it). The mistake is using `model_copy` for an operation that processes external user input — the withdrawal amount comes from outside, so it should be validated.

**3. The security implication:**

An attacker sends:
```
POST /accounts/ACC-001/withdraw
Body: {"amount": 999999999.0}
```

`apply_withdrawal` computes `100.0 - 999999999.0 = -999999899.0` and calls `model_copy` with this value. No ValidationError is raised. The returned `BankAccount` instance has `balance = -999999899.0`. If your database layer calls `session.add(updated_account)` and `session.commit()`, the negative balance is persisted. The attacker has created unlimited debt.

**4. Safe fix:**

```python
def apply_withdrawal(account: BankAccount, amount: float) -> BankAccount:
    new_balance = account.balance - amount
    # Construct a new instance — goes through __init__, triggers Field(ge=0)
    return BankAccount(
        account_id=account.account_id,
        owner=account.owner,
        balance=new_balance,    # ValidationError here if negative
        is_frozen=account.is_frozen,
    )

# Alternative — round-trip through model_validate
def apply_withdrawal(account: BankAccount, amount: float) -> BankAccount:
    new_balance = account.balance - amount
    return BankAccount.model_validate({
        **account.model_dump(),
        "balance": new_balance
    })
```

Both approaches pass through Pydantic's validation pipeline because they use `__init__` (or `model_validate`, which calls `__init__` internally).

---

## Solution 1.4: The First-Match Trap

**1. Exact output:**

```
A type:   FreeShipping
A cost:   FIELD MISSING
A method: 'paid'

B type:   FreeShipping

C type:   FreeShipping
C cost:   FIELD MISSING
C method: 'free'
```

All three orders produce `FreeShipping`. Including Scenario A, where `cost=9.99` was explicitly provided.

**2. Scenario A's Union resolution, step by step:**

Input: `{"method": "paid", "cost": 9.99, "estimated_days": 3}`

```
Step 1: Try FreeShipping
  FreeShipping has:
    method: str = "free"       ← has a default; any str value accepted
    estimated_days: int = 7    ← has a default; int 3 is valid

  Does input satisfy FreeShipping's required fields? Yes — there are NONE.
  "method": "paid"      → accepted as str, overrides default "free"
  "estimated_days": 3   → accepted as int
  "cost": 9.99          → extra field, silently dropped (extra="ignore" default)

  FreeShipping instantiates successfully.

Step 2: PaidShipping is NEVER tried.
```

Result: `FreeShipping(method="paid", estimated_days=3)`. The `cost` field does not exist on `FreeShipping`; `9.99` is gone.

**3. The missing cost:**

`order_a.shipping.cost` raises `AttributeError: 'FreeShipping' object has no attribute 'cost'`. The developer expected `9.99` — they get a crash. And the business logic has silently treated a paid order as free shipping.

**4. The fix — discriminated union with `Literal` fields:**

```python
from typing import Literal, Union
from pydantic import BaseModel, Field

class FreeShipping(BaseModel):
    method: Literal["free"]         # Only exactly "free" accepted
    estimated_days: int = 7

class PaidShipping(BaseModel):
    method: Literal["paid"]         # Only exactly "paid" accepted
    cost: float
    estimated_days: int

class Order(BaseModel):
    item_name: str
    shipping: Union[FreeShipping, PaidShipping] = Field(
        discriminator="method"      # Read "method" first; jump to correct model
    )
```

With `Literal["free"]` and `Literal["paid"]`, the `method` field is no longer "any string" — it is exactly one specific value. The discriminator tells Pydantic to read `method` first and select the model that declared that literal, making selection O(1) and deterministic.

An unknown value (e.g., `method="express"`) immediately raises:
```
ValidationError: method → Input should be 'free' or 'paid'
```

**The key insight:** `str` means "any string is valid here — it's just a label." `Literal["free"]` means "exactly this string, and nothing else — it identifies the type." That semantic shift transforms a vague description into a discriminating key.

---

# LEVEL 2: DEBUG, FIX & VERIFY

## Exercise 2.1: The Validator Saboteur

**Context:** A developer built a contact form submission handler. It never crashes in development where they always provide all fields in tests. QA reports two separate problems: (1) submitter names are corrupted in the database, and (2) the endpoint intermittently returns a `500 Internal Server Error` when the `phone` field is omitted from the request body. The developer is confused — phone is an optional field, so omitting it should be fine.

```python
from pydantic import BaseModel, Field, field_validator, model_validator

class ContactForm(BaseModel):
    full_name: str = Field(min_length=2)
    email: str
    phone: str | None = None    # Optional — can be omitted
    message: str = Field(min_length=10)

    @field_validator("full_name")
    @classmethod
    def clean_full_name(cls, v: str) -> str:
        v.strip().title()           # Processes the value

    @field_validator("email")
    @classmethod
    def normalize_email(cls, v: str) -> str:
        return v.strip().lower()    # Returns the value

    @model_validator(mode="before")
    @classmethod
    def normalize_phone(cls, data: dict) -> dict:
        """Strip non-digit characters from phone number if provided."""
        raw_phone = data["phone"]   # Access the phone value
        if raw_phone is not None:
            data["phone"] = "".join(c for c in raw_phone if c.isdigit())
        return data

# Developer's test — always passes, no complaints
form = ContactForm(
    full_name="  alice smith  ",
    email="  ALICE@Example.COM  ",
    phone="(555) 123-4567",
    message="I have a question about your service."
)
print(f"name:  {form.full_name!r}")
print(f"email: {form.email!r}")
print(f"phone: {form.phone!r}")
```

---

**Task 2a: Identify the Flaws**

There are **two distinct bugs**. The developer's test above hides both because it always provides all fields. Without running the code:

1. What is the actual value of `form.full_name` when the code runs? Explain what value was expected vs. what is produced.
2. Why does the endpoint crash with a `500` when `phone` is omitted from a real API request? Which specific line causes the error, and what type of exception is it?

---

**Task 2b: Explain Each Mechanism**

1. **Bug 1 (full_name):** Walk through exactly what `clean_full_name` does with input `"  alice smith  "`. What does `v.strip().title()` return? What value does the function ultimately return? Why does the field end up corrupted?

2. **Bug 2 (phone):** In `model_validator(mode="before")`, the `data` parameter is the raw input dictionary — it contains **only the keys explicitly provided by the caller**. Defaults have not been applied yet. What happens when a caller omits `phone` and the validator executes `data["phone"]`? Why doesn't the `phone: str | None = None` default prevent this?

---

**Task 2c: Instrument to Prove Diagnosis**

Add print statements to expose each bug without fixing it:

```python
@field_validator("full_name")
@classmethod
def clean_full_name(cls, v: str) -> str:
    processed = v.strip().title()
    print(f"DEBUG full_name: input={v!r}, processed={processed!r}, returning=???")
    v.strip().title()  # original broken line — leave as-is

@model_validator(mode="before")
@classmethod
def normalize_phone(cls, data: dict) -> dict:
    print(f"DEBUG phone: keys in data = {list(data.keys())}")
    raw_phone = data["phone"]
    # ... rest unchanged
```

Show the output from the developer's passing test, then show the output (including the exception) when `phone` is omitted.

---

**Task 2d: Fix the Code**

Fix both bugs. Requirements:
- `full_name="  alice smith  "` → `form.full_name == "Alice Smith"` (stripped and title-cased)
- Omitting `phone` from the input must work without error
- When `phone="(555) 123-4567"` is provided, it should normalize to `"5551234567"`
- `email` normalization must continue to work

---

**Task 2e: Write Verification Tests**

Write two tests — one per bug — that **fail on the original code** and **pass on the fixed code**:

```python
from pydantic import ValidationError

def test_full_name_is_normalized():
    form = ContactForm(
        full_name="  alice smith  ",
        email="a@b.com",
        message="This is a test message."
    )
    assert form.full_name == "Alice Smith", (
        f"Expected 'Alice Smith', got {form.full_name!r}"
    )

def test_phone_is_optional():
    # Must not raise any exception when phone is omitted
    form = ContactForm(
        full_name="Bob Jones",
        email="bob@example.com",
        message="Testing optional phone field."
    )
    assert form.phone is None
```

---

## Exercise 2.2: The Phantom Tree

**Context:** A developer built a recursive comment thread model. They wrote tests — all pass. The code ships to production. Within hours, the on-call engineer is woken up: the "total engagement" feature is crashing for any post that has replies to replies.

```python
from pydantic import BaseModel
from datetime import datetime

class Comment(BaseModel):
    id: int
    author: str
    text: str
    created_at: datetime
    upvotes: int = 0
    replies: list["Comment"] = []

    def total_upvotes(self) -> int:
        """Sum of upvotes for this comment and all nested replies."""
        return self.upvotes + sum(r.total_upvotes() for r in self.replies)

    def find_by_id(self, target_id: int) -> "Comment | None":
        """Find a comment by ID anywhere in the thread."""
        for reply in self.replies:
            if reply.id == target_id:
                return reply
            found = reply.find_by_id(target_id)
            if found:
                return found
        return None

# Developer's tests — ALL PASS in CI
def test_single_comment():
    c = Comment(id=1, author="alice", text="Hello",
                created_at=datetime(2025, 1, 1), upvotes=5)
    assert c.total_upvotes() == 5       # No replies, just counts own upvotes
    assert c.find_by_id(999) is None
    print("test_single_comment: PASS")

def test_comment_with_one_reply():
    c = Comment(
        id=1, author="alice", text="Hello",
        created_at=datetime(2025, 1, 1), upvotes=5,
        replies=[{"id": 2, "author": "bob", "text": "Hi",
                  "created_at": "2025-01-02T00:00:00", "upvotes": 3}]
    )
    assert c.upvotes == 5                # Only checks own upvotes — doesn't call total_upvotes!
    print("test_comment_with_one_reply: PASS")

test_single_comment()
test_comment_with_one_reply()
print("All tests passed — shipping to production")

# --- PRODUCTION CRASH ---
thread = Comment(
    id=1, author="alice", text="Great article!",
    created_at=datetime(2025, 1, 1), upvotes=10,
    replies=[
        {
            "id": 2, "author": "bob", "text": "Agreed!",
            "created_at": "2025-01-02T00:00:00", "upvotes": 5,
            "replies": [
                {"id": 3, "author": "carol", "text": "Me too!",
                 "created_at": "2025-01-03T00:00:00", "upvotes": 2}
            ]
        }
    ]
)

total = thread.total_upvotes()   # ← CRASHES HERE in production
```

---

**Task 2a: Identify the Flaw**

1. Why do both developer tests pass? Specifically, what does `test_comment_with_one_reply` fail to verify that would have caught the bug?
2. What is `type(thread.replies[0])` at runtime? Is it `Comment` or `dict`? State your answer before any instrumentation.
3. Where exactly does `total_upvotes()` crash and with what exception type?

---

**Task 2b: Explain the Mechanism**

1. What does Python do when it encounters `list["Comment"]` inside the `Comment` class body at definition time? Why is the type unresolved at that moment?
2. Without `model_rebuild()`, when Pydantic validates the input `replies=[{"id": 2, ...}]`, what does it do with each dict in the list? Why doesn't Pydantic raise an error when the dicts enter the model?
3. Walk through what happens when `total_upvotes()` calls `sum(r.total_upvotes() for r in self.replies)` on a `Comment` whose `replies` contains plain dicts.

---

**Task 2c: Instrument to Prove Diagnosis**

Add the following diagnostic block before the crashing line and show the exact output:

```python
print(f"replies[0] type:        {type(thread.replies[0]).__name__}")
print(f"replies[0] is dict:     {isinstance(thread.replies[0], dict)}")
print(f"replies[0] is Comment:  {isinstance(thread.replies[0], Comment)}")

try:
    _ = thread.replies[0].total_upvotes()
except AttributeError as e:
    print(f"AttributeError: {e}")
```

---

**Task 2d: Fix the Code**

Fix the `Comment` model so that:
- All nested replies are validated as `Comment` instances at any depth
- `total_upvotes()` works for arbitrarily deep nesting
- `find_by_id()` works correctly
- The fix adds exactly **one line** to the class definition (outside the class body)

---

**Task 2e: Write Verification Tests**

Write tests that **fail on the broken code** and **pass on the fixed code**:

```python
def test_nested_reply_is_comment_instance():
    c = Comment(
        id=1, author="alice", text="Hello",
        created_at=datetime(2025, 1, 1),
        replies=[{"id": 2, "author": "bob", "text": "Hi",
                  "created_at": "2025-01-02T00:00:00"}]
    )
    # Fails without model_rebuild() — replies[0] is a dict, not Comment
    assert isinstance(c.replies[0], Comment), (
        f"Expected Comment instance, got {type(c.replies[0]).__name__}"
    )

def test_total_upvotes_with_nested_replies():
    thread = Comment(
        id=1, author="alice", text="Root", created_at=datetime(2025, 1, 1),
        upvotes=10,
        replies=[
            {"id": 2, "author": "bob", "text": "Reply",
             "created_at": "2025-01-02T00:00:00", "upvotes": 5,
             "replies": [
                 {"id": 3, "author": "carol", "text": "Nested",
                  "created_at": "2025-01-03T00:00:00", "upvotes": 2}
             ]}
        ]
    )
    assert thread.total_upvotes() == 17  # 10 + 5 + 2

def test_find_by_id_finds_nested_comment():
    thread = Comment(
        id=1, author="alice", text="Root", created_at=datetime(2025, 1, 1),
        replies=[
            {"id": 2, "author": "bob", "text": "Reply",
             "created_at": "2025-01-02T00:00:00",
             "replies": [
                 {"id": 3, "author": "carol", "text": "Nested",
                  "created_at": "2025-01-03T00:00:00"}
             ]}
        ]
    )
    found = thread.find_by_id(3)
    assert found is not None
    assert isinstance(found, Comment)
    assert found.author == "carol"
```

---

# Level 2 Solutions

## Solution 2.1: The Validator Saboteur

**Task 2a: The Two Bugs**

1. **`form.full_name`** is `None`. The `clean_full_name` validator computes `v.strip().title()` but the result is never captured or returned. The function returns `None` implicitly, and Pydantic sets the field to `None`.

2. **The 500 when `phone` is omitted**: `data["phone"]` raises `KeyError: 'phone'`. In `model_validator(mode="before")`, `data` is the raw input dictionary — it only contains the keys the caller explicitly provided. `phone: str | None = None` is a field *default*, but defaults are applied by Pydantic's field resolution step, which runs **after** `mode="before"` validators. At the time `normalize_phone` runs, the dict is the raw user input with no defaults injected. `data["phone"]` therefore raises `KeyError`.

---

**Task 2b: Mechanisms**

**Bug 1:**
```
Call: clean_full_name(cls, v="  alice smith  ")

v.strip().title()
  → "  alice smith  ".strip() returns "alice smith" (new object, discarded)
  → "alice smith".title()    returns "Alice Smith"  (new object, discarded)
  → Expression result is never assigned or returned

Function ends: implicit return None
Pydantic sets form.full_name = None
```

The field type is `str`, but Pydantic trusts the validator's return value after `mode="after"` processing. The type is not re-checked.

**Bug 2:**

`model_validator(mode="before")` runs before field validation and before defaults are applied. The `data` dict is exactly `{"full_name": "...", "email": "...", "message": "..."}` when phone is omitted. There is no `"phone"` key. `data["phone"]` raises `KeyError`.

The `= None` default on the field declaration is an instruction to Pydantic's field-resolution logic. That logic runs *after* all `mode="before"` validators complete. The two steps are strictly ordered and do not share state.

---

**Task 2c: Instrumented output**

With all fields provided (developer's test):
```
DEBUG full_name: input='  alice smith  ', processed='Alice Smith', returning=???
DEBUG phone: keys in data = ['full_name', 'email', 'phone', 'message']
name:  None                ← Bug 1: None despite valid input
email: 'alice@example.com'
phone: '5551234567'
```

With phone omitted (production case):
```
DEBUG phone: keys in data = ['full_name', 'email', 'message']
KeyError: 'phone'
```

---

**Task 2d: Fixed code:**

```python
class ContactForm(BaseModel):
    full_name: str = Field(min_length=2)
    email: str
    phone: str | None = None
    message: str = Field(min_length=10)

    @field_validator("full_name")
    @classmethod
    def clean_full_name(cls, v: str) -> str:
        return v.strip().title()       # FIX 1: return the result

    @field_validator("email")
    @classmethod
    def normalize_email(cls, v: str) -> str:
        return v.strip().lower()

    @model_validator(mode="before")
    @classmethod
    def normalize_phone(cls, data: dict) -> dict:
        raw_phone = data.get("phone")  # FIX 2: .get() returns None if key absent
        if raw_phone is not None:
            data["phone"] = "".join(c for c in raw_phone if c.isdigit())
        return data
```

---

**Task 2e: Verification tests:**

Both tests below fail on the original, pass on the fixed version.

```python
def test_full_name_is_normalized():
    form = ContactForm(
        full_name="  alice smith  ",
        email="a@b.com",
        message="This is a test message."
    )
    assert form.full_name == "Alice Smith", (
        f"Expected 'Alice Smith', got {form.full_name!r}"
    )

def test_phone_is_optional():
    form = ContactForm(
        full_name="Bob Jones",
        email="bob@example.com",
        message="Testing optional phone field."
    )
    assert form.phone is None
```

---

## Solution 2.2: The Phantom Tree

**Task 2a: The Flaw**

1. `test_single_comment` passes because there are no replies — `total_upvotes()` and `find_by_id()` never iterate over `self.replies`. `test_comment_with_one_reply` passes because it only asserts `c.upvotes == 5`, which accesses the top-level field directly. It **never calls** `total_upvotes()` or `find_by_id()` on the nested structure — the very operations that require replies to be `Comment` instances, not dicts.

2. `type(thread.replies[0])` is `dict`. Without `model_rebuild()`, the `list["Comment"]` annotation is unresolved at validation time, so Pydantic stores the raw dict as-is.

3. `total_upvotes()` calls `sum(r.total_upvotes() for r in self.replies)`. The first `r` is a `dict`. `dict` has no `.total_upvotes()` method. Python raises `AttributeError: 'dict' object has no attribute 'total_upvotes'`.

---

**Task 2b: The Mechanism**

1. When Python reads `replies: list["Comment"] = []`, it stores the annotation as the string `"Comment"` (a forward reference). The class `Comment` is not yet fully defined — the class body is still being executed. Pydantic records this as an unresolved forward reference.

2. Without `model_rebuild()`, Pydantic never resolves `"Comment"` to the actual `Comment` class. When it encounters `replies=[{...}]` at validation time, it has no schema for the list's element type. Rather than raising an error, it stores the dicts as-is — they pass through without validation because the element type's schema was never built.

3. `total_upvotes()` on `thread`:
   - `self.upvotes` = 10 (OK)
   - `for r in self.replies`: `r` is `{"id": 2, "author": "bob", ..., "replies": [...]}` — a plain dict
   - `r.total_upvotes()` → `AttributeError: 'dict' object has no attribute 'total_upvotes'`
   - Exception propagates; production crashes.

---

**Task 2c: Diagnostic output:**

```
replies[0] type:        dict
replies[0] is dict:     True
replies[0] is Comment:  False
AttributeError: 'dict' object has no attribute 'total_upvotes'
```

---

**Task 2d: Fixed code:**

```python
class Comment(BaseModel):
    id: int
    author: str
    text: str
    created_at: datetime
    upvotes: int = 0
    replies: list["Comment"] = []

    def total_upvotes(self) -> int:
        return self.upvotes + sum(r.total_upvotes() for r in self.replies)

    def find_by_id(self, target_id: int) -> "Comment | None":
        for reply in self.replies:
            if reply.id == target_id:
                return reply
            found = reply.find_by_id(target_id)
            if found:
                return found
        return None

Comment.model_rebuild()    # ← The one required line: resolves "Comment" → Comment
```

---

**Task 2e: Verification tests:**

The three tests in the exercise body all fail on the original (no `model_rebuild`) and pass on the fixed version. The key assertion is:
```python
assert isinstance(c.replies[0], Comment)
```
Without `model_rebuild()`, this fails because `replies[0]` is a `dict`. With it, `replies[0]` is a fully validated `Comment` instance.

---

# LEVEL 2.5: EXTEND EXISTING CODE

**Context:** The following working `Product` model and test suite are provided. You must add three extensions in sequence. **After each extension, all previously passing tests must still pass.**

```python
# product.py — working baseline
from pydantic import BaseModel, Field, ConfigDict

class Product(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True)

    name: str = Field(min_length=1, max_length=100)
    price: float = Field(gt=0, description="Price in USD")
    category: str
    stock: int = Field(default=0, ge=0)
    description: str = ""
    sale_price: float | None = Field(default=None, gt=0)
```

```python
# test_product.py — provided tests (MUST CONTINUE TO PASS)
import pytest
from pydantic import ValidationError

def test_product_creation():
    p = Product(name="Widget", price=9.99, category="tools")
    assert p.name == "Widget"
    assert p.price == 9.99
    assert p.stock == 0

def test_whitespace_stripped():
    p = Product(name="  Widget  ", price=9.99, category="  tools  ")
    assert p.name == "Widget"
    assert p.category == "tools"

def test_negative_price_rejected():
    with pytest.raises(ValidationError):
        Product(name="Widget", price=-1.0, category="tools")

def test_zero_sale_price_rejected():
    with pytest.raises(ValidationError):
        Product(name="Widget", price=10.0, category="tools", sale_price=0)

def test_sale_price_is_optional():
    p = Product(name="Widget", price=9.99, category="tools")
    assert p.sale_price is None
```

---

### Extension 1: Computed Discount Percentage

Add a `discount_pct` computed field to `Product` that:
- Returns `0.0` when `sale_price` is `None`
- Returns the percentage discount when `sale_price` is set: `(price - sale_price) / price * 100`, rounded to 2 decimal places

```python
# Tests for Extension 1 — add to test_product.py
def test_discount_pct_with_sale_price():
    p = Product(name="Widget", price=10.0, category="tools", sale_price=8.0)
    assert p.discount_pct == 20.0

def test_discount_pct_without_sale_price():
    p = Product(name="Widget", price=10.0, category="tools")
    assert p.discount_pct == 0.0

def test_discount_pct_in_model_dump():
    p = Product(name="Widget", price=10.0, category="tools", sale_price=7.5)
    data = p.model_dump()
    assert "discount_pct" in data
    assert data["discount_pct"] == 25.0
```

---

### Extension 2: Cross-Field Sale Price Validation

Add a `@model_validator` that rejects a `sale_price` that is greater than or equal to the regular `price`. The validator must handle the case where `sale_price` is `None` without raising an error.

```python
# Tests for Extension 2 — add to test_product.py
def test_sale_price_above_price_rejected():
    with pytest.raises(ValidationError) as exc_info:
        Product(name="Widget", price=10.0, category="tools", sale_price=15.0)
    assert "sale_price must be less than price" in str(exc_info.value)

def test_sale_price_equal_to_price_rejected():
    with pytest.raises(ValidationError):
        Product(name="Widget", price=10.0, category="tools", sale_price=10.0)

def test_no_sale_price_still_valid():
    # Extension 2 must NOT break the case where sale_price is None
    p = Product(name="Widget", price=10.0, category="tools")
    assert p.sale_price is None
```

---

### Extension 3: Bulk Product ID Validation

Add a **module-level** `TypeAdapter` named `ProductIdListAdapter` typed as `TypeAdapter(list[PositiveInt])`, and a function `parse_product_ids(raw: str) -> list[int]` that:
- Splits the input string on commas
- Strips whitespace from each part
- Validates the resulting list using `ProductIdListAdapter`
- Returns the validated list of positive integers

The adapter **must** be defined at module level, not inside the function.

```python
# Tests for Extension 3 — add to test_product.py
def test_parse_product_ids_valid():
    result = parse_product_ids("1, 2, 3, 4")
    assert result == [1, 2, 3, 4]

def test_parse_product_ids_rejects_zero():
    with pytest.raises(Exception):   # ValidationError
        parse_product_ids("1,0,3")

def test_parse_product_ids_rejects_negative():
    with pytest.raises(Exception):
        parse_product_ids("1,-5,3")

def test_parse_product_ids_rejects_non_numeric():
    with pytest.raises(Exception):
        parse_product_ids("1,abc,3")

def test_adapter_is_module_level():
    import product as product_module
    import inspect
    # Verify the adapter is NOT defined inside the function
    source = inspect.getsource(product_module.parse_product_ids)
    assert "TypeAdapter" not in source, (
        "TypeAdapter must be defined at module level, not inside the function"
    )
```

---

# Level 2.5 Solutions

## Solution — All Three Extensions

```python
# product.py — complete with all three extensions

from typing import Annotated
from pydantic import BaseModel, Field, ConfigDict, TypeAdapter, PositiveInt
from pydantic import computed_field, model_validator

class Product(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True)

    name: str = Field(min_length=1, max_length=100)
    price: float = Field(gt=0, description="Price in USD")
    category: str
    stock: int = Field(default=0, ge=0)
    description: str = ""
    sale_price: float | None = Field(default=None, gt=0)

    # EXTENSION 1: Computed field — recalculated on every access
    @computed_field
    @property
    def discount_pct(self) -> float:
        if self.sale_price is None:
            return 0.0
        return round((self.price - self.sale_price) / self.price * 100, 2)

    # EXTENSION 2: Cross-field model validator
    @model_validator(mode="after")
    def sale_price_must_be_less_than_price(self) -> "Product":
        if self.sale_price is not None and self.sale_price >= self.price:
            raise ValueError(
                f"sale_price must be less than price "
                f"(got sale_price={self.sale_price}, price={self.price})"
            )
        return self


# EXTENSION 3: Module-level TypeAdapter — built once, reused on every call
ProductIdListAdapter = TypeAdapter(list[PositiveInt])

def parse_product_ids(raw: str) -> list[int]:
    parts = [part.strip() for part in raw.split(",")]
    return ProductIdListAdapter.validate_python(parts)
```

**Why each extension was designed this way:**

**Extension 1 — `@computed_field`:** The `discount_pct` depends on `price` and `sale_price`. It must not be stored separately (it would become stale if either field changed on a mutable model). `@computed_field` with `@property` declares it as a derived value that appears in `model_dump()` and the OpenAPI schema automatically. The return type annotation (`-> float`) is required — omitting it raises `PydanticUserError`.

**Extension 2 — `@model_validator(mode="after")`:** This validation crosses two fields (`sale_price` and `price`). A `@field_validator` on `sale_price` alone cannot see `price`. The mode is `"after"` because both fields need to be validated (correctly typed as floats) before the comparison is meaningful. The guard `if self.sale_price is not None` is required because `sale_price` is optional — failing to guard would raise `TypeError: '<' not supported between instances of 'NoneType' and 'float'` whenever `sale_price` is omitted.

**Extension 3 — Module-level TypeAdapter:** `TypeAdapter(list[PositiveInt])` builds a complete Pydantic validation schema internally. This is an expensive operation — comparable to defining a small `BaseModel`. Placing it inside `parse_product_ids` would rebuild the entire schema on every function call. At module level, it is built exactly once on import and reused for every call.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## The Seat Booking System

**Background:** A concert venue needs an API for booking seats. The venue has three seat categories — Standard, Premium, and VIP — each with different fields. Bookings must be representable in three forms: creation input, API response, and partial updates. Bulk booking cancellation must accept a comma-separated list of booking IDs as a query parameter.

**Your task is to implement all models and functions so that the provided test suite passes.**

---

### Constraints (These make the naive approach impossible)

**Constraint A — Discriminated Union is mandatory.** You must use a `Literal` field named `"category"` as the discriminator. A plain `Union` without a discriminator will fail the test that verifies unknown category values produce clear errors naming all valid options.

**Constraint B — The Read model must not expose `internal_cost`.** The venue's internal cost per seat is stored for accounting. It must never appear in any API response. The test verifies this by attempting to find `internal_cost` in `model_dump()` output.

**Constraint C — PATCH must use `exclude_unset=True`.** The test for partial updates sends only `notes` in the update body and verifies that `seat_count` is unchanged. Without `exclude_unset=True`, your PATCH logic will reset `seat_count` to `None`.

**Constraint D — `TypeAdapter` must be defined at module level.** The test inspects the source code of your parse function to verify `TypeAdapter` does not appear inside it.

---

### Provided Test Suite

Save this as `test_booking.py` and run `pytest test_booking.py` to verify your implementation:

```python
# test_booking.py — provided; do not modify

import inspect
import pytest
from pydantic import ValidationError
from datetime import datetime, timezone

# Import everything from your implementation file
from booking import (
    StandardSeat, PremiumSeat, VIPSeat,
    BookingCreate, BookingRead, BookingUpdate,
    parse_booking_ids,
)

NOW = datetime(2025, 6, 1, 12, 0, tzinfo=timezone.utc)


# ─── 3a: Core discriminated union ────────────────────────────────────────────

def test_standard_seat_validates():
    seat = StandardSeat(category="standard", price=50.0, row="G", seat_number=14)
    assert seat.category == "standard"
    assert seat.price == 50.0

def test_premium_seat_validates():
    seat = PremiumSeat(
        category="premium", price=120.0, row="C", seat_number=5,
        includes_drink=True
    )
    assert seat.includes_drink is True

def test_vip_seat_validates():
    seat = VIPSeat(
        category="vip", price=350.0, row="A", seat_number=1,
        lounge_access=True, meet_and_greet=False
    )
    assert seat.lounge_access is True

def test_unknown_category_raises_clear_error():
    """
    Discriminated union must produce an error listing valid category values.
    A plain Union without discriminator fails this test because the error message
    is produced by trying each model sequentially and is hard to read.
    """
    with pytest.raises(ValidationError) as exc_info:
        BookingCreate(
            seat={"category": "platinum", "price": 500.0,
                  "row": "A", "seat_number": 1},
            seat_count=2,
            booker_email="test@test.com",
        )
    # The error must immediately identify which values are valid
    errors = exc_info.value.errors()
    error_input_options = str(errors)
    assert any(
        cat in error_input_options
        for cat in ["standard", "premium", "vip"]
    ), f"Error should name valid categories; got: {errors}"

def test_standard_seat_rejected_without_row():
    with pytest.raises(ValidationError):
        StandardSeat(category="standard", price=50.0, seat_number=1)


# ─── 3b: Create / Read / Update pattern ──────────────────────────────────────

def test_booking_read_excludes_internal_cost():
    """internal_cost must NEVER appear in any serialized Read model output."""
    read = BookingRead(
        id=1,
        seat=StandardSeat(category="standard", price=50.0,
                          row="G", seat_number=14),
        seat_count=2,
        booker_email="alice@example.com",
        internal_cost=35.0,    # stored internally
        confirmed_at=NOW,
        notes=None,
    )
    data = read.model_dump()
    assert "internal_cost" not in data, (
        f"internal_cost must be excluded from Read output; got keys: {list(data.keys())}"
    )

def test_booking_update_all_fields_optional():
    """BookingUpdate must be creatable with zero fields — all are optional."""
    update = BookingUpdate()
    assert update.notes is None
    assert update.seat_count is None

def test_partial_update_preserves_other_fields():
    """
    When updating only `notes`, `seat_count` must remain unchanged.
    This test fails if you do `existing.model_copy(update=update.model_dump())`
    instead of `update.model_dump(exclude_unset=True)`.
    """
    existing = {
        "id": 1,
        "seat": {"category": "standard", "price": 50.0,
                 "row": "G", "seat_number": 14},
        "seat_count": 4,
        "booker_email": "alice@example.com",
        "internal_cost": 35.0,
        "confirmed_at": NOW.isoformat(),
        "notes": None,
    }

    update = BookingUpdate(notes="Window seat preferred")
    # Simulate your PATCH logic: merge only what was explicitly set
    update_data = update.model_dump(exclude_unset=True)
    merged = {**existing, **update_data}

    result = BookingRead.model_validate(merged)
    assert result.seat_count == 4, (
        f"seat_count must be preserved; got {result.seat_count}"
    )
    assert result.notes == "Window seat preferred"


# ─── 3c: TypeAdapter bulk validation ─────────────────────────────────────────

def test_parse_booking_ids_valid():
    result = parse_booking_ids("1, 2, 3, 100")
    assert result == [1, 2, 3, 100]

def test_parse_booking_ids_rejects_zero():
    with pytest.raises(Exception):
        parse_booking_ids("1,0,3")

def test_parse_booking_ids_rejects_non_integer():
    with pytest.raises(Exception):
        parse_booking_ids("1,two,3")

def test_adapter_not_inside_function():
    """TypeAdapter must be defined at module level, not inside parse_booking_ids."""
    import booking
    source = inspect.getsource(booking.parse_booking_ids)
    assert "TypeAdapter" not in source, (
        "TypeAdapter must be at module level, not inside the function. "
        "Building it inside a function rebuilds the schema on every call."
    )
```

---

### Implementation Workflow

**3a — Implement the discriminated union seat models and `BookingCreate`** so that the `3a` tests pass.

**3b — Add `BookingRead` and `BookingUpdate`** so the `3b` tests pass. Apply `from_attributes=True` to `BookingRead` since it will eventually be built from ORM objects. Make `internal_cost` invisible in the Read output.

**3c — Add the module-level `TypeAdapter` and `parse_booking_ids`** so the `3c` tests pass.

---

# Level 3 Solution

```python
# booking.py — full reference implementation

from typing import Literal, Union, Annotated
from datetime import datetime
from pydantic import (
    BaseModel, Field, ConfigDict,
    TypeAdapter, PositiveInt
)


# ─── 3a: Seat models with discriminated union ─────────────────────────────────

class StandardSeat(BaseModel):
    category: Literal["standard"]
    price: float = Field(gt=0)
    row: str = Field(min_length=1)
    seat_number: int = Field(gt=0)

class PremiumSeat(BaseModel):
    category: Literal["premium"]
    price: float = Field(gt=0)
    row: str = Field(min_length=1)
    seat_number: int = Field(gt=0)
    includes_drink: bool = False

class VIPSeat(BaseModel):
    category: Literal["vip"]
    price: float = Field(gt=0)
    row: str = Field(min_length=1)
    seat_number: int = Field(gt=0)
    lounge_access: bool = True
    meet_and_greet: bool = False

# The discriminated union type alias — usable anywhere a seat is expected
SeatType = Annotated[
    Union[StandardSeat, PremiumSeat, VIPSeat],
    Field(discriminator="category")
]


# ─── 3b: Create / Read / Update models ───────────────────────────────────────

class BookingCreate(BaseModel):
    """What the client sends to create a booking. No generated fields."""
    seat: SeatType
    seat_count: int = Field(gt=0, default=1)
    booker_email: str
    notes: str | None = None
    # No: id (DB generates), confirmed_at (server generates), internal_cost (never from client)


class BookingRead(BaseModel):
    """What the server returns. internal_cost is stored but never exposed."""
    model_config = ConfigDict(from_attributes=True)

    id: int
    seat: SeatType
    seat_count: int
    booker_email: str
    confirmed_at: datetime
    notes: str | None = None
    internal_cost: float = Field(exclude=True)   # Stored, never serialized
    # Note: Field(exclude=True) is equivalent to model_dump(exclude={"internal_cost"})
    # but is baked into the model — it can never be accidentally included.


class BookingUpdate(BaseModel):
    """What the client sends to partially update a booking. All optional."""
    seat_count: int | None = Field(default=None, gt=0)
    notes: str | None = None
    # seat and booker_email are immutable after creation — not in Update


# ─── 3c: Module-level TypeAdapter ────────────────────────────────────────────

BookingIdListAdapter = TypeAdapter(list[PositiveInt])

def parse_booking_ids(raw: str) -> list[int]:
    """Parse a comma-separated string of booking IDs into a validated list."""
    parts = [part.strip() for part in raw.split(",")]
    return BookingIdListAdapter.validate_python(parts)
```

**Why each constraint forced a specific decision:**

**Constraint A** forced `Literal` fields. Without `Literal["standard"]`, the discriminator has no distinct value to key on. `Literal` transforms a descriptive string into a type-selecting identifier — the model that declares `category: Literal["vip"]` is the only valid target when `category="vip"` arrives. The error message lists all valid literal values automatically.

**Constraint B** forced `Field(exclude=True)` on `internal_cost` in `BookingRead`. An alternative is `model_dump(exclude={"internal_cost"})` at every call site — but this is a per-call decision, meaning a single forgotten call site leaks the field. `Field(exclude=True)` is a **model-level guarantee**: `internal_cost` cannot appear in serialized output regardless of how `model_dump()` is called.

**Constraint C** forced `update.model_dump(exclude_unset=True)`. Without it, `BookingUpdate()` produces `{"seat_count": None, "notes": None}`, and merging this with the existing record resets `seat_count` to `None`. `exclude_unset=True` returns only the fields the client explicitly provided — the only semantically correct behavior for `PATCH`.

**Constraint D** forced module-level placement. `TypeAdapter(list[PositiveInt])` compiles a complete Rust-backed validation schema. Inside a function, this schema is rebuilt on every call — in a high-traffic endpoint processing thousands of requests, this becomes a measurable bottleneck. At module level, the schema is built once on application startup.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## The Distinction

**Prompt:** Describe **two concrete scenarios** where using Pydantic's `BaseModel` validation is actively worse than not using it. For each scenario, write a minimal code snippet demonstrating the failure mode.

Your answer must:
1. Name the specific property of Pydantic that causes the problem
2. Show code where the naive "just use Pydantic" instinct leads to measurably worse outcomes
3. Describe what to use instead

---

# Level 4 Solutions

### Scenario 1: High-Frequency Internal Pipelines With Trusted Data

**The property causing the problem:** Pydantic's validation pipeline (type coercion, constraint checking, error collection) has non-trivial overhead. This overhead is justified when validating external, untrusted input. It is pure waste when processing data that was already validated at entry and is flowing through internal pipeline stages.

```python
# ❌ WRONG for this use case — Pydantic in an inner loop on trusted data
from pydantic import BaseModel

class InternalMetric(BaseModel):
    sensor_id: int
    value: float
    timestamp: float

def process_sensor_batch(raw_rows: list[tuple]) -> list[float]:
    """
    Called 50,000 times per second.
    raw_rows come directly from a typed database cursor —
    the types are already guaranteed correct by the DB schema.
    """
    results = []
    for row in raw_rows:
        metric = InternalMetric(
            sensor_id=row[0],    # Already int from DB
            value=row[1],        # Already float from DB
            timestamp=row[2]     # Already float from DB
        )
        results.append(metric.value * 1.05)
    return results
```

```python
# ✅ CORRECT for this use case — plain dataclass, zero overhead
from dataclasses import dataclass

@dataclass
class InternalMetric:
    sensor_id: int
    value: float
    timestamp: float

def process_sensor_batch(raw_rows: list[tuple]) -> list[float]:
    results = []
    for row in raw_rows:
        metric = InternalMetric(sensor_id=row[0], value=row[1], timestamp=row[2])
        results.append(metric.value * 1.05)
    return results
```

**The rule:** Validate at the **boundary** (API input, file parsing, database read, message queue consumer). Inside the boundary, trust the data and use lightweight data carriers (`dataclass`, `NamedTuple`, or plain objects). Pydantic's purpose is to enforce the contract *at the gate*, not to re-check every door inside the building.

---

### Scenario 2: Truly Dynamic Schema — Unknown Fields at Definition Time

**The property causing the problem:** `BaseModel` requires field names and types to be declared at class definition time. When the schema itself is unknown until runtime (user-defined forms, plugin-provided configurations, dynamic JSON structures), a fixed `BaseModel` is the wrong tool.

```python
# ❌ WRONG — trying to force Pydantic where the schema is dynamic
from pydantic import BaseModel, ConfigDict

# A survey platform lets admins define questions with arbitrary types.
# We cannot declare these at class definition time:

class SurveyResponse(BaseModel):
    model_config = ConfigDict(extra="allow")
    # We can't declare: q1: str, q2: int, q3: float
    # because the survey designer chooses these at runtime.
    pass

# This "works" but we get ZERO validation on the dynamic fields.
# Any value is accepted for any field — the entire purpose of Pydantic is lost.
response = SurveyResponse(**{"favorite_color": 99999, "age": "not a number"})
print(response.model_dump())
# {"favorite_color": 99999, "age": "not a number"}
# Nothing was validated — this is just a dict with extra steps.
```

```python
# ✅ CORRECT — TypedDict or plain dict with explicit per-field validation
from pydantic import TypeAdapter

# Validate each field against its per-question schema
def validate_survey_response(
    response: dict,
    schema: dict[str, type]   # {"age": int, "color": str, ...}
) -> dict:
    validated = {}
    for field_name, field_type in schema.items():
        adapter = TypeAdapter(field_type)
        raw_value = response.get(field_name)
        validated[field_name] = adapter.validate_python(raw_value)
    return validated
```

**The rule:** Pydantic's strength is the *static*, declared schema — it gives you IDE autocomplete, type checker support, and OpenAPI generation precisely because the schema is fixed. When you need truly dynamic schemas, Pydantic's static nature is a liability. Use `TypeAdapter` for per-value validation, or a schema-agnostic validation library.

---

# LEVEL 5: THE TEACH BACK

## The Synthesis

**Scenario:** A junior developer on your team pushes a PR with the following explanation in the PR description:

> *"I noticed we have three almost-identical models for User: `UserCreate`, `UserRead`, and `UserUpdate`. This is obvious code duplication. They all have the same fields: `username`, `email`, `password_hash`, `role`, `created_at`, `id`. I've refactored them into a single `User` model with all fields optional so it can handle all three cases. This removes ~60 lines of duplication and simplifies the codebase."*

Their unified model looks like this:

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int | None = None
    username: str | None = None
    email: str | None = None
    password_hash: str | None = None
    role: str | None = None
    created_at: str | None = None
```

They've wired it up so this single model handles:
- `POST /register` — registration input
- `GET /users/{id}` — profile response
- `PATCH /users/{id}` — partial update

Write a response to this PR explaining:
1. Why this approach introduces a **security vulnerability**, with a concrete exploit
2. Why it **corrupts the API documentation** in a specific, demonstrable way
3. Why the "duplication" they're removing is not incidental boilerplate but **deliberate semantic separation**
4. The correct pattern with the minimal explanation of why each model must be distinct

---

# Level 5 Solution

---

**PR Review — Major Issues: Do Not Merge**

I understand the impulse — three models with similar fields looks like repetition, and deleting 60 lines feels like improvement. But this change introduces a security hole that goes to production silently, corrupts the Swagger documentation for every consumer of this API, and eliminates validation that is currently protecting us in three different ways. Let me walk through each issue concretely.

---

**1. The Security Vulnerability**

With the unified model as the `response_model` for `GET /users/{id}`:

```python
@app.get("/users/{user_id}", response_model=User)
async def get_user(user_id: int, db_user: DBUser = Depends(get_db_user)):
    return User.model_validate(db_user)
```

The `User` model has `password_hash: str | None = None`. When `model_validate` reads from the database object, `password_hash` is populated — it's a real field on the DB record. `model_dump()` includes it. FastAPI serializes it into the JSON response.

Any authenticated user can now call `GET /users/{id}` and receive the Argon2id hash of every user's password in the response body. A targeted offline dictionary attack against those hashes becomes possible. The `UserRead` model existed specifically to make this impossible by simply not declaring `password_hash` as a field.

The critical distinction: **a field that isn't declared in a model cannot accidentally appear in its output**. `model_dump(exclude={"password_hash"})` in every handler is a per-call defense — one forgotten endpoint and the hash leaks. A separate `UserRead` model with no `password_hash` field is a model-level guarantee that cannot be forgotten.

---

**2. The Documentation Corruption**

FastAPI generates OpenAPI documentation from `response_model`. With the unified model as the response for `GET /users/{id}`, Swagger UI now shows:

```
GET /users/{id}
Response 200:
  id:            integer | null
  username:      string | null
  email:         string | null
  password_hash: string | null   ← appears in public docs
  role:          string | null
  created_at:    string | null
```

Every client — the frontend team, third-party integrators, anyone with access to the docs — can now see `password_hash` in the documented response schema. The field name alone tells attackers how password storage is implemented.

Beyond the security issue: every field is `| null`. The frontend developer calling this endpoint cannot tell which fields are guaranteed to be present. Is `username` always there? What does `null` mean here — the user has no username, or just that it's optional? The `UserRead` model communicates: *"These fields are always present in a successful response."* The unified model communicates nothing.

---

**3. The Duplication Is Deliberate**

`UserCreate`, `UserRead`, and `UserUpdate` are not three copies of the same model. They model three distinct real-world contracts:

A **registration input** (`UserCreate`) carries a plaintext `password` (never `password_hash`), has no `id` (the database generates it), has no `created_at` (the server sets it), and has every required field mandatory — you cannot register with missing credentials.

A **profile response** (`UserRead`) has `id` and `created_at` (they now exist), carries no password or hash (the client never needs this), and has every field present — a response with missing fields is a server bug, not a client error.

A **partial update** (`UserUpdate`) has every field optional — you can update only the email, only the username, or only the role. Making everything `| None` in the unified model collapses three precise contracts into one vague contract that accurately describes none of them.

The fields *look* similar because they describe the same domain entity. But their required/optional structure, their included/excluded fields, and their semantic meaning differ completely across the three operations.

---

**4. The Correct Pattern**

```python
# Shared constraints declared once — DRY where it actually applies
class UserBase(BaseModel):
    username: str = Field(min_length=3, max_length=30)
    email: str

# Registration: client sends plaintext password; no id or hash
class UserCreate(UserBase):
    password: str = Field(min_length=8)

# Response: server-generated fields present; password never included
class UserRead(UserBase):
    model_config = ConfigDict(from_attributes=True)
    id: int
    role: str
    created_at: datetime

# Partial update: everything optional; role update excluded from self-service
class UserUpdate(BaseModel):
    username: str | None = Field(default=None, min_length=3, max_length=30)
    email: str | None = None
```

`UserBase` is where the actual DRY principle applies — `username` and `email` constraints are declared once. `UserCreate`, `UserRead`, and `UserUpdate` are semantically distinct and remain separate. The `password_hash` field does not appear in any of these three models — it lives only in the SQLAlchemy ORM model, never in the Pydantic layer.

This is not 60 lines of duplication. It is 60 lines of explicit, verifiable, documented contract.