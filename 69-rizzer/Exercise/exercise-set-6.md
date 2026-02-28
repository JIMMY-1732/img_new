# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "Testing is just running my code again with `assert` statements. A fixture is a helper function that sets up test data, a mock is something I reach for when calling a real API is inconvenient, and if my coverage number is high, my code is well-tested."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---|---|
| 1 | `scope="module"` makes the fixture faster without changing test behaviour | `scope="module"` shares ONE object across ALL tests in that file. Mutations from test 1 are visible in test 2 — isolation is destroyed |
| 2 | `MagicMock` is a universal fake that works anywhere | `MagicMock` cannot be `await`ed. Replacing an `async def` dependency requires `AsyncMock`, or the test raises `TypeError` at the `await` step |
| 3 | Patch where a function is **defined** | You must patch where the function is **looked up** (imported and used). Patching the definition namespace leaves every other binding untouched |
| 4 | Mocking all dependencies makes tests "more isolated" | Mocking the function **under test** — not its dependencies — removes the code being tested entirely. You are now testing the mock |
| 5 | High line coverage proves the code is correct | Coverage measures which lines were **executed**. `assert isinstance(result, float)` gives 100% coverage and proves nothing about correctness |
| 6 | `pytest.raises(SomeError)` validates the right error occurred | Without `match=`, it accepts any `SomeError` raised from any line in the block, including setup-code bugs and unrelated exceptions |

**The Prediction Gap:**
When shown a `scope="module"` fixture returning a mutable dict, followed by three tests that each write into it, students with the naive model predict every test sees `{"events": [], "count": 0}`. The actual behaviour is a single, accumulating dict handed to every test in the file — all mutations persist.

---

## The Main Purpose (Essence)

**Core Insight:** A test is not "extra code that confirms my code runs." It is a **machine-checkable specification of behaviour** that must be isolated between runs, must target the real code (not a mock of it), and must be designed to *falsify* assumptions — not merely confirm them.

**The Problems Being Solved:**

| Pattern | Naive Use | Correct Use |
|---|---|---|
| Fixture scope | `scope="session"` on everything for speed | `scope="function"` (default) for isolated mutable state; broader scopes only for genuinely shared, immutable resources |
| `MagicMock` vs `AsyncMock` | `MagicMock` for every dependency | `AsyncMock` for every `async def` you replace |
| What to mock | Mock everything, including the tested function | Mock only at the boundary: external HTTP, time, file system, database |
| Patch location | `patch("lib.original_module.func")` | `patch("src.your_module.func")` — where the name is actually used |
| What to assert | `assert isinstance(result, float)` | Assert the specific, correct computed value |

**The Key Distinction:**
`scope="function"` → fresh object for **every** test. `scope="module"` → **one shared** object for every test in the file. These look identical in the test parameter list. The contamination is completely silent.

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behaviour | Root Cause |
|---|---|---|---|
| "Each test gets a fresh fixture" | `scope="module"` dict is `{}` for every test | Accumulated mutations from all previous tests | Fixture object is reused, not recreated |
| "`MagicMock` is a universal fake" | Mock works, test passes | `TypeError: object list can't be used in 'await' expression` | `await` needs a coroutine; `MagicMock()` does not produce one |
| "Patch where the function lives" | `patch("secrets.token_hex")` intercepts all calls | Real function is called; mock never activates | `from secrets import token_hex` binds an independent local name; only THAT name is in scope |
| "More mocking = more isolation" | Mocking `Invoice.total` catches all total bugs | Bug in `Invoice.total` is invisible; test passes regardless | The mock replaces the buggy property with a hardcoded number |
| "Coverage % = correctness %" | 100% coverage + all passes = working code | Hidden crash on `quantity=0`, silent wrong answer on sign flip | Coverage measures execution; assertions measure correctness |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1 — The Module-Scope Time Bomb

**Context:** A team building the notification pipeline for the mini-project decides to use `scope="module"` on a shared audit fixture to "speed up the test suite."

```python
# tests/conftest.py
import pytest

@pytest.fixture(scope="module")
def dispatch_log():
    """Shared log tracking dispatch events across tests."""
    return {"sent": [], "failed": [], "total": 0}
```

```python
# tests/test_dispatch.py

def test_first_notification_dispatched(dispatch_log):
    dispatch_log["sent"].append("welcome_email")
    dispatch_log["total"] += 1

    assert len(dispatch_log["sent"]) == 1
    assert dispatch_log["total"] == 1

def test_failed_notification_recorded(dispatch_log):
    dispatch_log["failed"].append("sms_timeout")
    dispatch_log["total"] += 1

    assert len(dispatch_log["failed"]) == 1   # ← Line A
    assert dispatch_log["total"] == 1          # ← Line B

def test_log_is_clean_for_new_scenario(dispatch_log):
    assert dispatch_log["sent"] == []          # ← Line C
    assert dispatch_log["total"] == 0          # ← Line D
    assert "failed" in dispatch_log            # ← Line E
```

**Questions:**

1. **Predict** whether each marked assertion (A, B, C, D, E) passes or fails. For every failing assertion, write the actual value being compared on the left-hand side.

2. **Explain** why `test_log_is_clean_for_new_scenario` does not see a clean log, even though no test has explicitly shared data with it. Use the term "object identity" in your explanation.

3. **Mental Trace:** How many `dispatch_log` dict objects exist in memory across the entire test run? At exactly what moment is the fixture object created? When is it destroyed?

4. **The Ordering Trap:** A teammate argues: "Tests in the same file are logically related, so sharing state between them is fine — if `test_log_is_clean` depends on order, we just run it first." Explain precisely why this argument makes the test suite less reliable, not more.

---

## Exercise 1.2 — The Phantom Patch

**Context:** A developer tests the API key generator for the mini-project authentication module. They use `patch` and `return_value`. The test fails — but for a reason that is easy to misdiagnose.

```python
# src/auth.py
from secrets import token_hex  # ← from-import, not `import secrets`

def generate_api_key() -> str:
    """Generate a cryptographically secure API key."""
    raw = token_hex(32)
    return f"sk_{raw}"

def create_user_credentials(username: str) -> dict[str, str]:
    """Create user credentials with a generated API key."""
    api_key = generate_api_key()
    return {
        "username": username,
        "api_key": api_key,
        "key_prefix": api_key[:5],
    }
```

```python
# tests/test_auth.py
from unittest.mock import patch
from src.auth import create_user_credentials

def test_credentials_contain_fixed_key():
    with patch("secrets.token_hex") as mock_token:  # ← patch target
        mock_token.return_value = "ff" * 32          # 64 hex chars

        result = create_user_credentials("alice")

        assert result["api_key"] == "sk_" + "ff" * 32
        assert result["key_prefix"] == "sk_ff"
        mock_token.assert_called_once_with(32)
```

**Questions:**

1. **Predict the failure.** The test fails. Does it fail on the first `assert`, or does it fail on `mock_token.assert_called_once_with(32)`? What does `result["api_key"]` actually contain?

2. **Explain the import mechanism.** When Python executes `from secrets import token_hex` inside `src/auth.py`, draw the resulting name-binding picture. Why does patching `secrets.token_hex` (the attribute on the `secrets` module) not affect what `generate_api_key()` calls?

3. **Diagnose the teammate's argument.** A teammate insists: "`generate_api_key` calls `token_hex`, which IS `secrets.token_hex`, so patching `secrets.token_hex` must intercept the call." Find the precise flaw in this reasoning.

4. **Fix** the patch target with a single-word change. Explain why the corrected target works when `patch("secrets.token_hex")` does not.

---

## Exercise 1.3 — The 100% Coverage Illusion

**Context:** A pricing module for a B2B platform. The test suite reports 100% line coverage and all tests pass.

```python
# src/pricing.py
from dataclasses import dataclass

@dataclass
class PricingTier:
    name: str
    min_quantity: int
    discount_percent: float

TIERS = [
    PricingTier(name="standard",  min_quantity=1,  discount_percent=0.0),
    PricingTier(name="bulk",      min_quantity=10, discount_percent=10.0),
    PricingTier(name="wholesale", min_quantity=50, discount_percent=20.0),
]

def get_best_tier(quantity: int) -> PricingTier:
    """Return the highest tier the quantity qualifies for."""
    qualifying = [t for t in TIERS if t.min_quantity <= quantity]
    return max(qualifying, key=lambda t: t.min_quantity)

def calculate_price(quantity: int, unit_price: float) -> float:
    """Calculate total price after best applicable discount."""
    tier = get_best_tier(quantity)
    discount = unit_price * (tier.discount_percent / 100)
    return (unit_price - discount) * quantity
```

```python
# tests/test_pricing.py — Coverage: 100% ✓  All tests: PASS ✓
from src.pricing import calculate_price, get_best_tier

def test_pricing_module_executes():
    result = calculate_price(quantity=50, unit_price=10.0)
    assert isinstance(result, float)    # ← asserts only that a float came back

def test_tier_lookup_returns_something():
    tier = get_best_tier(quantity=10)
    assert tier is not None             # ← asserts only that something was returned
```

**Questions:**

1. **Predict:** What does `pytest --cov=src.pricing --cov-report=term-missing` report for line coverage? What appears in the "Missing" column?

2. **Find the silent crash.** Neither test calls `get_best_tier(quantity=0)`. Trace step-by-step what happens when `quantity=0` is passed. Name the exception and the exact line it originates from.

3. **Find the undetected wrong answer.** Introduce this single-character bug:
   ```python
   return (unit_price + discount) * quantity  # + instead of -
   ```
   Do the existing tests catch this? What does the test output say?

4. **State the diagnostic principle.** In one precise sentence: what does line coverage measure, and what does it explicitly NOT measure?

5. **Rewrite** the test suite with at least four assertions that would catch the crash from question 2 AND the wrong-sign bug from question 3. Use `pytest.mark.parametrize` for at least one test.

---

## Level 1 Solutions

### Solution 1.1

**1. Predictions:**

| Line | Assertion | Result | Actual LHS value |
|---|---|---|---|
| A | `len(dispatch_log["failed"]) == 1` | **PASS** | `["sms_timeout"]` → length 1 ✓ |
| B | `dispatch_log["total"] == 1` | **FAIL** | `2` (incremented in test 1) |
| C | `dispatch_log["sent"] == []` | **FAIL** | `["welcome_email"]` (appended in test 1) |
| D | `dispatch_log["total"] == 0` | **FAIL** | `2` |
| E | `"failed" in dispatch_log` | **PASS** | key exists from fixture construction ✓ |

**2. Explain — object identity:**
`scope="module"` creates exactly one dict object (call it `D`) before the first test in the file. Every test receives **the same object reference** — `id(dispatch_log)` is identical in all three tests. When test 1 calls `.append()` and `+= 1`, it mutates `D`. When test 2 receives `dispatch_log`, it receives `D` — which already contains `["welcome_email"]` and `total=1`. There is no reset, no copy, and no recreation between tests. "Shared" means shared memory address.

**3. Mental Trace:**
```
Time 0: pytest loads conftest.py. No fixture constructed yet.
Time 1: pytest collects test_dispatch.py. Still no construction.

Time 2: First test runs — test_first_notification_dispatched.
         FIRST request for dispatch_log in this module.
         → Fixture function is called → ONE dict created: D = {"sent":[], "failed":[], "total":0}
         → D passed to test.
         → Mutations: D["sent"]=["welcome_email"], D["total"]=1

Time 3: test_failed_notification_recorded runs.
         dispatch_log requested again. scope="module": D returned. NOT recreated.
         → Mutations: D["failed"]=["sms_timeout"], D["total"]=2

Time 4: test_log_is_clean_for_new_scenario runs.
         dispatch_log requested again. Same D.
         → D is now: {"sent":["welcome_email"], "failed":["sms_timeout"], "total":2}
         → All "clean" assertions fail.

Total dict objects created: 1. Created before test 1. Destroyed after the last test in the module.
```

**4. The Ordering Trap Refutation:**
If `test_log_is_clean_for_new_scenario` passes when run in isolation (`pytest tests/test_dispatch.py::test_log_is_clean_for_new_scenario`) but fails in the full suite, then its result depends on execution order. pytest does not guarantee order by default. Anyone running a single test, running tests in a different order, or using `pytest-xdist` for parallel execution will get inconsistent results. This is a **flaky test** — non-deterministic and order-dependent. It is worse than no test, because it creates false confidence when run in isolation. The fix is to remove `scope="module"` and let each test receive a fresh fixture.

---

### Solution 1.2

**1. The failure:**
The first assertion fails:
```
AssertionError: assert 'sk_3fa8b2...' == 'sk_' + 'ff' * 32
```
`result["api_key"]` contains a real, randomly-generated key — the mock was never activated. `mock_token.assert_called_once_with(32)` would also raise `AssertionError: Expected call: mock(32) / Not called` if reached, confirming the mock was never invoked at all.

**2. The import mechanism:**
When Python executes `from secrets import token_hex` inside `src/auth.py`:
```
secrets module namespace       src.auth module namespace
──────────────────────         ──────────────────────────────
secrets.token_hex  ──────────► src.auth.token_hex  ──► <function token_hex at 0x...>
```
`src.auth` now holds its **own independent name** `token_hex`, pointing directly to the function object. `patch("secrets.token_hex")` replaces the `token_hex` attribute on the `secrets` module object. But `src.auth.token_hex` is a **separate binding** — it was created at import time and points to the original function. Changing `secrets.token_hex` does not touch `src.auth.token_hex`.

**3. The flaw in the teammate's argument:**
"Which IS `secrets.token_hex`" was true at import time — both names pointed to the same function object. But they are **independent pointers**. Patching `secrets.token_hex` reassigns one pointer; it does not affect the other. The function object at `0x...` still exists and `src.auth.token_hex` still points to it. Python names are bindings, not aliases that update automatically.

**4. The fix:**
```python
with patch("src.auth.token_hex") as mock_token:  # ← Changed from secrets.token_hex
```
This replaces `src.auth.token_hex` — the exact name that `generate_api_key()` looks up at runtime. Now when `generate_api_key` executes `raw = token_hex(32)`, Python finds the mock in `src.auth`'s namespace and calls it instead of the real function.

---

### Solution 1.3

**1. Coverage:**
```
Name             Stmts   Miss  Cover   Missing
----------------------------------------------
src/pricing.py      15      0   100%
```
Coverage is 100%. The "Missing" column is empty. Both tests pass with no output about uncovered lines — because every line in the module runs when `calculate_price(50, 10.0)` is called. The list comprehension, the `max()`, and the arithmetic all execute. Coverage is satisfied. Correctness is not.

**2. Silent crash:**
```
Call: get_best_tier(quantity=0)
→ qualifying = [t for t in TIERS if t.min_quantity <= 0]
  TIERS has min_quantity values of 1, 10, 50. None satisfy <= 0.
→ qualifying = []   (empty list)
→ max([], key=...)
→ ValueError: max() arg is an empty sequence
```
The exception originates from `return max(qualifying, key=lambda t: t.min_quantity)` — the last line of `get_best_tier`. The test suite never calls with `quantity <= 0`, so this crash is completely invisible.

**3. Undetected wrong answer:**
With `(unit_price + discount) * quantity`:
- `calculate_price(50, 10.0)` → tier=wholesale (20%), discount=2.0 → `(10.0 + 2.0) * 50 = 600.0`
- `assert isinstance(600.0, float)` → **PASS**

The test suite reports 2 passed. The sign flip is silent. Revenue would be inflated by up to 40% in production.

**4. The Diagnostic Principle:**
Line coverage measures which lines were **executed** during the test run; it does not measure whether those lines produced **correct output** or were **tested against inputs that expose their failure modes**.

**5. Rewritten test suite:**
```python
import pytest
from src.pricing import calculate_price, get_best_tier

def test_standard_tier_full_price():
    """1-9 items: no discount, full price."""
    assert calculate_price(quantity=1,  unit_price=10.0) == 10.0
    assert calculate_price(quantity=9,  unit_price=10.0) == 90.0

def test_bulk_tier_ten_percent_off():
    """10-49 items: 10% off. Catches the +/- sign bug."""
    # 10 × 10.0 × 0.90 = 90.0. With the bug: 10 × 12.0 = 120.0 → FAIL
    assert calculate_price(quantity=10, unit_price=10.0) == 90.0

def test_wholesale_tier_twenty_percent_off():
    """50+ items: 20% off."""
    # 50 × 10.0 × 0.80 = 400.0. With the bug: 50 × 12.0 = 600.0 → FAIL
    assert calculate_price(quantity=50, unit_price=10.0) == 400.0

@pytest.mark.parametrize("quantity", [0, -1, -100])
def test_zero_or_negative_quantity_raises(quantity: int):
    """Catches the silent crash from get_best_tier with empty qualifying list."""
    with pytest.raises(ValueError):
        calculate_price(quantity=quantity, unit_price=10.0)
```

---

# LEVEL 2: DEBUG, FIX & VERIFY

## Exercise 2.1 — The Async Invisible Wall

**Context:** A developer is mocking the API layer for the mini-project's concurrent news fetcher. The test is correctly structured: `patch` is used, `return_value` is set, and `@pytest.mark.asyncio` is applied. The test still raises a `TypeError` before any assertion.

```python
# src/news_fetcher.py
import httpx
from dataclasses import dataclass

@dataclass
class Article:
    title: str
    source: str

async def fetch_headlines(api_key: str, category: str = "technology") -> list[Article]:
    """Fetch top headlines from a news API."""
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.get(
            "https://newsapi.org/v2/top-headlines",
            params={"apiKey": api_key, "category": category},
        )
        response.raise_for_status()
        data = response.json()
        return [
            Article(title=item["title"], source=item["source"]["name"])
            for item in data.get("articles", [])
        ]

async def get_tech_summary(api_key: str) -> str:
    """Return a formatted summary of the top three tech headlines."""
    articles = await fetch_headlines(api_key, category="technology")
    if not articles:
        return "No tech news available."
    return "Top stories: " + " | ".join(a.title for a in articles[:3])
```

```python
# tests/test_news_fetcher.py — FLAWED
import pytest
from unittest.mock import patch
from src.news_fetcher import get_tech_summary, Article

@pytest.mark.asyncio
async def test_tech_summary_shows_top_three():
    mock_articles = [
        Article(title="Python 4.0 Released", source="TechCrunch"),
        Article(title="AI Sets New Record",  source="Wired"),
        Article(title="Rust Surpasses Go",   source="HackerNews"),
    ]

    with patch("src.news_fetcher.fetch_headlines") as mock_fetch:
        mock_fetch.return_value = mock_articles   # ← the mistake

        result = await get_tech_summary("fake-key")

        assert "Python 4.0 Released" in result
        assert "AI Sets New Record" in result
```

---

**Task 2a: Identify the Flaw**

The test raises before reaching any `assert`. Without running the code, identify the exact line in `get_tech_summary` that triggers the error and write the exact error message.

---

**Task 2b: Explain the Mechanism**

1. When `patch` is applied to `fetch_headlines` without `new_callable=AsyncMock`, what type of object replaces it?
2. Break down `articles = await fetch_headlines(api_key, category="technology")` into two distinct steps: the **call step** and the **await step**. What does each step produce?
3. Why does setting `mock_fetch.return_value = mock_articles` not resolve the problem?

---

**Task 2c: Instrument to Prove Diagnosis**

Without running the test, add two diagnostic `print` statements inside the `with patch(...)` block — before calling `get_tech_summary` — that reveal the type of `mock_fetch` and the type of `mock_fetch.return_value`. State exactly what would be printed.

```python
with patch("src.news_fetcher.fetch_headlines") as mock_fetch:
    mock_fetch.return_value = mock_articles

    print(???)  # ← What does this print?
    print(???)  # ← And this?

    result = await get_tech_summary("fake-key")
```

---

**Task 2d: Fix the Code**

Fix the test with a one-import, one-keyword change. Do not restructure the test — change only what is strictly necessary.

---

**Task 2e: Write a Verification Test**

Write `test_tech_summary_empty_response` that:
- Mocks `fetch_headlines` with the correct mock type
- Makes it return an empty list
- Asserts the exact return value is `"No tech news available."`
- Verifies `fetch_headlines` was called with `category="technology"`

---

## Exercise 2.2 — The Test That Tests the Mock

**Context:** A developer tests `format_invoice_summary` by reaching for `MagicMock` to "isolate" the test from the `Invoice` class. The test passes. It is fundamentally broken.

```python
# src/invoice.py
from dataclasses import dataclass, field

@dataclass
class LineItem:
    description: str
    quantity: int
    unit_price: float

@dataclass
class Invoice:
    client_name: str
    items: list[LineItem] = field(default_factory=list)
    tax_rate: float = 0.1

    @property
    def subtotal(self) -> float:
        return sum(item.quantity * item.unit_price for item in self.items)

    @property
    def tax_amount(self) -> float:
        return round(self.subtotal * self.tax_rate, 2)

    @property
    def total(self) -> float:
        return round(self.subtotal + self.tax_amount, 2)

    def add_item(self, description: str, quantity: int, unit_price: float) -> None:
        if quantity <= 0:
            raise ValueError(f"Quantity must be positive, got {quantity}")
        if unit_price < 0:
            raise ValueError(f"Price cannot be negative, got {unit_price}")
        self.items.append(LineItem(description, quantity, unit_price))

def format_invoice_summary(invoice: Invoice) -> str:
    """Format a human-readable invoice summary."""
    lines = [f"Invoice for: {invoice.client_name}"]
    for item in invoice.items:
        lines.append(f"  {item.description}: {item.quantity} × ${item.unit_price:.2f}")
    lines.append(f"Subtotal: ${invoice.subtotal:.2f}")
    lines.append(f"Tax ({invoice.tax_rate * 100:.0f}%): ${invoice.tax_amount:.2f}")
    lines.append(f"Total: ${invoice.total:.2f}")
    return "\n".join(lines)
```

```python
# tests/test_invoice.py — FLAWED
from unittest.mock import MagicMock
from src.invoice import format_invoice_summary

def test_invoice_summary_is_formatted():
    mock_invoice = MagicMock()
    mock_invoice.client_name = "Acme Corp"
    mock_invoice.tax_rate    = 0.1
    mock_invoice.items = [
        MagicMock(description="Widget", quantity=5, unit_price=10.0)
    ]
    mock_invoice.subtotal    = 50.0   # manually set — not computed
    mock_invoice.tax_amount  = 5.0    # manually set — not computed
    mock_invoice.total       = 55.0   # manually set — not computed

    result = format_invoice_summary(mock_invoice)

    assert "Acme Corp" in result
    assert "$55.00" in result
    assert "Tax (10%):" in result
```

---

**Task 2a: Identify the Flaw**

Introduce this single-character bug into `Invoice.total`:
```python
@property
def total(self) -> float:
    return round(self.subtotal - self.tax_amount, 2)  # minus instead of plus
```
Does the test catch this bug? Why or why not? Name the precise mechanism by which the mock shields the bug.

---

**Task 2b: Explain the Mechanism**

Draw the dependency chain for a real `Invoice.total` versus `mock_invoice.total = 55.0`. Does `mock_invoice.total` call `mock_invoice.subtotal`? Does `mock_invoice.subtotal` iterate `mock_invoice.items`? What is the computational relationship between these values in the mock world versus the real world?

---

**Task 2c: Instrument to Prove Diagnosis**

Add the following lines before the assertions and state what they print:
```python
print(f"type of subtotal attribute: {type(mock_invoice.subtotal)}")
print(f"does subtotal compute from items?: {mock_invoice.subtotal is mock_invoice.items}")
```
What does the output reveal about the relationship between the mocked properties and the real computation chain?

---

**Task 2d: Fix the Code**

Rewrite the test using a real `Invoice` object — zero mocks. The test must:
- Assert the exact formatted subtotal, tax, and total lines
- Catch the wrong-sign bug from Task 2a (the test must **fail** with the bugged code and **pass** with correct code)

---

**Task 2e: Write Verification Tests**

Write two tests that expose behaviours a `MagicMock` can never detect:
1. `test_add_item_zero_quantity_raises` — verifies `add_item` raises `ValueError` with a message containing `"positive"` when `quantity=0` is passed. (Connects to Week 1 Lecture 2: custom exception hierarchies.)
2. `test_invoice_total_equals_subtotal_plus_tax` — verifies the mathematical relationship, catching the wrong-sign bug directly.

---

## Level 2 Solutions

### Solution 2.1

**Task 2a: The error:**
The crash occurs at this line in `get_tech_summary`:
```python
articles = await fetch_headlines(api_key, category="technology")
```
Specifically at the `await` step. The error message is:
```
TypeError: object list can't be used in 'await' expression
```

**Task 2b: The mechanism:**
1. Without `new_callable=AsyncMock`, `patch` replaces `fetch_headlines` with a plain `MagicMock`. `type(mock_fetch)` is `unittest.mock.MagicMock`.
2. Two steps:
   - **Call step:** `fetch_headlines(api_key, category="technology")` invokes the `MagicMock`, which returns `mock_fetch.return_value` — the Python `list` object `mock_articles`.
   - **Await step:** `await [Article(...), Article(...), Article(...)]` — Python checks if a `list` implements `__await__`. It does not. `TypeError` is raised.
3. `mock_fetch.return_value = mock_articles` controls what the mock **returns when called** — it does not make that return value **awaitable**. `await` needs a coroutine or an object implementing `__await__`. A list has neither. `AsyncMock`, by contrast, is specially designed so that calling it returns a coroutine-like object that yields `return_value` when awaited.

**Task 2c: Diagnostic output:**
```python
print(type(mock_fetch))               # <class 'unittest.mock.MagicMock'>
print(type(mock_fetch.return_value))  # <class 'list'>
```
`mock_fetch` is a `MagicMock`. Its `return_value` is a plain `list`. Neither is awaitable.

**Task 2d: The fix:**
```python
import pytest
from unittest.mock import patch, AsyncMock   # ← Add AsyncMock
from src.news_fetcher import get_tech_summary, Article

@pytest.mark.asyncio
async def test_tech_summary_shows_top_three():
    mock_articles = [
        Article(title="Python 4.0 Released", source="TechCrunch"),
        Article(title="AI Sets New Record",  source="Wired"),
        Article(title="Rust Surpasses Go",   source="HackerNews"),
    ]

    with patch(
        "src.news_fetcher.fetch_headlines",
        new_callable=AsyncMock,              # ← The only change
    ) as mock_fetch:
        mock_fetch.return_value = mock_articles

        result = await get_tech_summary("fake-key")

        assert "Python 4.0 Released" in result
        assert "AI Sets New Record" in result
```
With `AsyncMock`, calling `mock_fetch(...)` returns a coroutine. `await` that coroutine yields `mock_articles`. Both the call step and the await step succeed.

**Task 2e: Verification test:**
```python
@pytest.mark.asyncio
async def test_tech_summary_empty_response():
    with patch(
        "src.news_fetcher.fetch_headlines",
        new_callable=AsyncMock,
    ) as mock_fetch:
        mock_fetch.return_value = []

        result = await get_tech_summary("fake-key")

        assert result == "No tech news available."
        mock_fetch.assert_called_once_with("fake-key", category="technology")
```

---

### Solution 2.2

**Task 2a: Bug detection:**
**No, the test does NOT catch the bug.** `format_invoice_summary` accesses `invoice.total`, which resolves to `mock_invoice.total` — a plain float `55.0` that was manually assigned. The real `Invoice.total` property with the `- self.tax_amount` bug is never called. The mock replaces the buggy property with a hardcoded number. The assertion `assert "$55.00" in result` passes because the mock said total is `55.0`, and `55.0` is what the mock returns. Production would compute `50.0 - 5.0 = 45.0`, but the test never knows.

**Task 2b: Dependency chain:**
```
Real Invoice:
  invoice.total  → calls invoice.subtotal property
                      → iterates invoice.items
                      → computes sum(qty × price)
                   → calls invoice.tax_amount property
                      → uses that subtotal value
                   → returns subtotal + tax_amount

MagicMock:
  mock_invoice.total      = 55.0   (Python float, assigned directly)
  mock_invoice.subtotal   = 50.0   (Python float, assigned directly)
  mock_invoice.tax_amount = 5.0    (Python float, assigned directly)
```
`mock_invoice.total` calls nothing. It does not call `mock_invoice.subtotal`. `mock_invoice.subtotal` does not iterate `mock_invoice.items`. The three values are independent, manually set constants with zero computational relationship to each other or to the `items` list.

**Task 2c: Diagnostic output:**
```python
print(f"type of subtotal attribute: {type(mock_invoice.subtotal)}")
# <class 'float'>   ← It was reassigned to a plain float, not a MagicMock

print(f"does subtotal compute from items?: {mock_invoice.subtotal is mock_invoice.items}")
# False   ← They are unrelated objects
```
`mock_invoice.subtotal` is just the float `50.0`. It has no connection to `mock_invoice.items`. The test is asserting the formatting of three independently invented numbers, not the output of any real computation.

**Task 2d: Fixed test:**
```python
from src.invoice import Invoice, format_invoice_summary

def test_invoice_summary_is_formatted():
    invoice = Invoice(client_name="Acme Corp")
    invoice.add_item("Widget", quantity=5, unit_price=10.0)
    # Real computation: subtotal = 5 × 10.0 = 50.0
    #                   tax      = 50.0 × 0.1 = 5.0
    #                   total    = 50.0 + 5.0 = 55.0

    result = format_invoice_summary(invoice)

    assert "Invoice for: Acme Corp" in result
    assert "Widget: 5 × $10.00" in result
    assert "Subtotal: $50.00" in result
    assert "Tax (10%): $5.00" in result
    assert "Total: $55.00" in result
    # With the minus bug: total = 50.0 - 5.0 = 45.0
    # assert "Total: $55.00" → FAIL ("Total: $45.00" found instead) ✓ Bug caught.
```

**Task 2e: Verification tests:**
```python
import pytest
from src.invoice import Invoice

def test_add_item_zero_quantity_raises():
    invoice = Invoice(client_name="Test Corp")
    with pytest.raises(ValueError, match="positive"):
        invoice.add_item("Widget", quantity=0, unit_price=10.0)

def test_invoice_total_equals_subtotal_plus_tax():
    invoice = Invoice(client_name="Test Corp", tax_rate=0.1)
    invoice.add_item("Item", quantity=10, unit_price=5.0)
    # subtotal = 50.0, tax = 5.0

    assert invoice.subtotal == 50.0
    assert invoice.tax_amount == 5.0
    assert invoice.total == 55.0   # FAILS with minus bug: 50.0 - 5.0 = 45.0 ≠ 55.0
```

---

# LEVEL 2.5: EXTEND EXISTING CODE

## Exercise 2.5 — Extending the Weather Client Test Suite

**Background:** The following suite is working and correct. All three tests pass. Your task is to extend it without modifying any existing code.

```python
# src/weather_client.py
import httpx
from dataclasses import dataclass

@dataclass
class WeatherData:
    city: str
    temperature: float
    description: str
    humidity: int

async def fetch_weather(api_key: str, city: str) -> WeatherData:
    """Fetch current weather from OpenWeatherMap."""
    async with httpx.AsyncClient(timeout=10.0) as client:
        response = await client.get(
            "https://api.openweathermap.org/data/2.5/weather",
            params={"q": city, "appid": api_key, "units": "metric"},
        )
        response.raise_for_status()
        data = response.json()
        return WeatherData(
            city=data["name"],
            temperature=data["main"]["temp"],
            description=data["weather"][0]["description"],
            humidity=data["main"]["humidity"],
        )

async def get_weather_report(api_key: str, city: str) -> str:
    """Return a formatted weather report string."""
    weather = await fetch_weather(api_key, city)
    return (
        f"{weather.city}: {weather.temperature:.1f}°C, "
        f"{weather.description} (humidity: {weather.humidity}%)"
    )
```

```python
# tests/test_weather_client.py — EXISTING PASSING TESTS (do not modify)
import pytest
from unittest.mock import patch, AsyncMock
from src.weather_client import get_weather_report, WeatherData

@pytest.fixture
def london_weather() -> WeatherData:
    return WeatherData(city="London", temperature=15.5,
                       description="partly cloudy", humidity=72)

@pytest.mark.asyncio
async def test_report_format(london_weather: WeatherData):
    with patch("src.weather_client.fetch_weather", new_callable=AsyncMock) as mock_fetch:
        mock_fetch.return_value = london_weather
        result = await get_weather_report("fake-key", "London")
    assert result == "London: 15.5°C, partly cloudy (humidity: 72%)"

@pytest.mark.asyncio
async def test_fetch_called_with_correct_api_key(london_weather: WeatherData):
    with patch("src.weather_client.fetch_weather", new_callable=AsyncMock) as mock_fetch:
        mock_fetch.return_value = london_weather
        await get_weather_report("my-real-key-123", "London")
    mock_fetch.assert_called_once_with("my-real-key-123", "London")

@pytest.mark.asyncio
async def test_below_zero_temperature_formatted():
    cold = WeatherData(city="Yakutsk", temperature=-42.3,
                       description="clear", humidity=30)
    with patch("src.weather_client.fetch_weather", new_callable=AsyncMock) as mock_fetch:
        mock_fetch.return_value = cold
        result = await get_weather_report("key", "Yakutsk")
    assert "-42.3°C" in result
```

**Constraint:** All three existing tests must continue passing after each extension. Do not touch any existing test code.

---

**Extension 1 — City Not Found:**
The OpenWeatherMap API returns HTTP 404 for unknown cities. When `raise_for_status()` is called, `httpx` raises `httpx.HTTPStatusError`. Currently `get_weather_report` does not catch this — the exception propagates to the caller.

Add a test `test_unknown_city_raises_http_status_error` that:
- Uses `side_effect` (not `return_value`) on an `AsyncMock` to raise `httpx.HTTPStatusError`
- Verifies the exception propagates out of `get_weather_report`
- Makes no real HTTP request

---

**Extension 2 — Factory Fixture:**
The existing `london_weather` fixture is inflexible. Any test needing a different temperature must construct `WeatherData` inline every time.

Add a `make_weather` factory fixture to `tests/conftest.py` that:
- Returns a callable with keyword defaults (`city="TestCity"`, `temperature=20.0`, `description="clear"`, `humidity=50`)
- Lets callers override only what matters

Then add `test_very_hot_city_in_report` that:
- Uses `make_weather(city="Dubai", temperature=43.8)`
- Asserts the output contains `"43.8°C"` and `"Dubai"`

---

**Extension 3 — Parametrized Temperatures:**
Add a single `@pytest.mark.parametrize` test `test_temperature_formatting` covering these four cases:

| Input temperature | Expected string in output |
|---|---|
| `0.0` | `"0.0°C"` |
| `-0.5` | `"-0.5°C"` |
| `100.0` | `"100.0°C"` |
| `20.5` | `"20.5°C"` |

Each case must mock `fetch_weather` correctly and assert the exact temperature string.

---

## Level 2.5 Solutions

### Extension 1

```python
# tests/test_weather_client.py (addition)
import httpx
from unittest.mock import MagicMock

@pytest.mark.asyncio
async def test_unknown_city_raises_http_status_error():
    mock_response = MagicMock()
    mock_response.status_code = 404
    error = httpx.HTTPStatusError(
        message="404 Not Found",
        request=MagicMock(),
        response=mock_response,
    )

    with patch("src.weather_client.fetch_weather", new_callable=AsyncMock) as mock_fetch:
        mock_fetch.side_effect = error  # ← raises on await, not returns

        with pytest.raises(httpx.HTTPStatusError):
            await get_weather_report("fake-key", "NotARealCity")
```

**Why `side_effect` not `return_value`:** `side_effect` on an `AsyncMock` makes the awaited coroutine raise the specified exception rather than yielding a value. `return_value` is for successful outcomes only; you cannot raise an exception via `return_value`.

### Extension 2

```python
# tests/conftest.py
import pytest
from src.weather_client import WeatherData

@pytest.fixture
def make_weather():
    """Factory fixture: create WeatherData with overrideable defaults."""
    def _factory(
        city: str = "TestCity",
        temperature: float = 20.0,
        description: str = "clear",
        humidity: int = 50,
    ) -> WeatherData:
        return WeatherData(
            city=city,
            temperature=temperature,
            description=description,
            humidity=humidity,
        )
    return _factory
```

```python
# tests/test_weather_client.py (addition)
@pytest.mark.asyncio
async def test_very_hot_city_in_report(make_weather):
    dubai = make_weather(city="Dubai", temperature=43.8)
    with patch("src.weather_client.fetch_weather", new_callable=AsyncMock) as mock_fetch:
        mock_fetch.return_value = dubai
        result = await get_weather_report("key", "Dubai")
    assert "43.8°C" in result
    assert "Dubai" in result
```

### Extension 3

```python
@pytest.mark.parametrize(
    "temperature, expected_str",
    [
        (0.0,   "0.0°C"),
        (-0.5,  "-0.5°C"),
        (100.0, "100.0°C"),
        (20.5,  "20.5°C"),
    ],
    ids=["zero", "sub_zero", "extreme_high", "mid_range"],
)
@pytest.mark.asyncio
async def test_temperature_formatting(
    temperature: float, expected_str: str, make_weather
):
    weather = make_weather(temperature=temperature)
    with patch("src.weather_client.fetch_weather", new_callable=AsyncMock) as mock_fetch:
        mock_fetch.return_value = weather
        result = await get_weather_report("key", weather.city)
    assert expected_str in result
```

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## Exercise 3 — Complete Test Suite for the Concurrent Aggregator (Mini-Project Core)

**Background:** This is the central module of the Week 2 mini-project. You will write the complete test suite from scratch.

```python
# src/aggregator.py
import asyncio
from dataclasses import dataclass
from datetime import datetime

import httpx

@dataclass
class NewsItem:
    title: str
    source: str

@dataclass
class WeatherSummary:
    city: str
    temp_celsius: float

@dataclass
class AggregatedReport:
    news: list[NewsItem]
    weather: WeatherSummary
    fetched_at: datetime

class APIKeyMissingError(Exception):
    """Raised when a required API key is absent or empty."""
    pass

async def fetch_news(api_key: str) -> list[NewsItem]:
    """Fetch technology headlines from NewsAPI."""
    if not api_key:
        raise APIKeyMissingError("NEWS_API_KEY is required but was not provided")
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.get(
            "https://newsapi.org/v2/top-headlines",
            params={"apiKey": api_key, "category": "technology"},
        )
        response.raise_for_status()
        data = response.json()
        return [
            NewsItem(title=a["title"], source=a["source"]["name"])
            for a in data.get("articles", [])
        ]

async def fetch_weather(api_key: str, city: str) -> WeatherSummary:
    """Fetch current weather from OpenWeatherMap."""
    if not api_key:
        raise APIKeyMissingError("WEATHER_API_KEY is required but was not provided")
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.get(
            "https://api.openweathermap.org/data/2.5/weather",
            params={"q": city, "appid": api_key, "units": "metric"},
        )
        response.raise_for_status()
        data = response.json()
        return WeatherSummary(city=data["name"], temp_celsius=data["main"]["temp"])

async def build_report(
    news_api_key: str,
    weather_api_key: str,
    city: str,
) -> AggregatedReport:
    """Fetch news and weather CONCURRENTLY and assemble a report."""
    news_task    = asyncio.create_task(fetch_news(news_api_key))
    weather_task = asyncio.create_task(fetch_weather(weather_api_key, city))

    news, weather = await asyncio.gather(news_task, weather_task)

    return AggregatedReport(
        news=news,
        weather=weather,
        fetched_at=datetime.now(),
    )
```

**Strict Constraints:**
1. **No real HTTP requests.** Every external call must be intercepted.
2. **`fetched_at` must be deterministic.** Use `freeze_time` to pin `datetime.now()`.
3. **No `time.sleep()`** anywhere in the test suite. All tests must complete in under 0.5 seconds.
4. **`APIKeyMissingError` must be tested** — connecting directly to Week 1 Lecture 2's custom exception pattern.
5. **The provided anchor test below must pass unmodified alongside your suite.**

```python
# PROVIDED: do not modify this test.
@pytest.mark.asyncio
async def test_report_has_fetched_at_timestamp():
    with patch("src.aggregator.fetch_news", new_callable=AsyncMock) as mn, \
         patch("src.aggregator.fetch_weather", new_callable=AsyncMock) as mw:
        mn.return_value = []
        mw.return_value = WeatherSummary(city="Berlin", temp_celsius=8.0)
        report = await build_report("k1", "k2", city="Berlin")
    assert isinstance(report.fetched_at, datetime)
```

---

**Task 3a — Write `conftest.py`:**

Create `tests/conftest.py` with three fixtures:
1. `make_news_item` — factory fixture returning a callable. Defaults: `title="Test Article"`, `source="TestSource"`.
2. `make_weather` — factory fixture returning a callable. Defaults: `city="TestCity"`, `temp_celsius=20.0`.
3. `fake_api_keys` — fixture returning `{"news": "fake-news-key", "weather": "fake-weather-key"}`.

---

**Task 3b — Test the Success Path:**

Write `test_build_report_success` that:
- Mocks both `fetch_news` and `fetch_weather` with `AsyncMock`
- Uses `freeze_time` as a context manager to pin time to `"2024-08-01 10:00:00"`
- Returns 3 `NewsItem`s and a Paris `WeatherSummary` from the respective mocks
- Asserts: `report.news` has exactly 3 items, `report.weather.city == "Paris"`, and `report.fetched_at == datetime(2024, 8, 1, 10, 0, 0)`
- Verifies `fetch_news` was called with the correct news key and `fetch_weather` with the correct weather key and `"Paris"`

---

**Task 3c — Test the Failure Path:**

Write `test_missing_news_api_key_raises`. Constraints:
- Pass `news_api_key=""` to `build_report`
- Do **NOT** mock `fetch_news` — let the real guard clause execute
- **DO** mock `fetch_weather` (to prevent any real HTTP attempt)
- Assert `APIKeyMissingError` is raised
- Assert the error message contains `"NEWS_API_KEY"` (connects to the `match=` pattern from the lecture)

---

**Task 3d — Parametrized City Test:**

Write a single `@pytest.mark.parametrize` test `test_weather_fetched_for_correct_city` that verifies `fetch_weather` is called with the correct city for: `"London"`, `"Tokyo"`, and `"São Paulo"`.

Use the `fake_api_keys` fixture. Mock both fetchers. For each city, assert `fetch_weather` was called exactly once and was called with the city string as its second argument.

---

## Level 3 Solutions

### Task 3a: `conftest.py`

```python
# tests/conftest.py
from __future__ import annotations
import pytest
from src.aggregator import NewsItem, WeatherSummary

@pytest.fixture
def make_news_item():
    """Factory fixture: create NewsItem objects with overrideable defaults."""
    def _factory(title: str = "Test Article", source: str = "TestSource") -> NewsItem:
        return NewsItem(title=title, source=source)
    return _factory

@pytest.fixture
def make_weather():
    """Factory fixture: create WeatherSummary objects with overrideable defaults."""
    def _factory(city: str = "TestCity", temp_celsius: float = 20.0) -> WeatherSummary:
        return WeatherSummary(city=city, temp_celsius=temp_celsius)
    return _factory

@pytest.fixture
def fake_api_keys() -> dict[str, str]:
    return {"news": "fake-news-key", "weather": "fake-weather-key"}
```

### Task 3b: Success path

```python
# tests/test_aggregator.py
import pytest
from datetime import datetime
from unittest.mock import patch, AsyncMock
from freezegun import freeze_time
from src.aggregator import (
    build_report, NewsItem, WeatherSummary, AggregatedReport, APIKeyMissingError
)

@pytest.mark.asyncio
async def test_build_report_success(make_news_item, make_weather, fake_api_keys):
    articles = [
        make_news_item(title="AI Breakthrough", source="TechCrunch"),
        make_news_item(title="Python 4.0",      source="InfoQ"),
        make_news_item(title="Rust Milestone",   source="Wired"),
    ]
    paris_weather = make_weather(city="Paris", temp_celsius=18.5)

    with patch("src.aggregator.fetch_news", new_callable=AsyncMock) as mock_news, \
         patch("src.aggregator.fetch_weather", new_callable=AsyncMock) as mock_weather:

        mock_news.return_value    = articles
        mock_weather.return_value = paris_weather

        with freeze_time("2024-08-01 10:00:00"):
            report = await build_report(
                news_api_key=fake_api_keys["news"],
                weather_api_key=fake_api_keys["weather"],
                city="Paris",
            )

    assert len(report.news) == 3
    assert report.weather.city == "Paris"
    assert report.fetched_at == datetime(2024, 8, 1, 10, 0, 0)

    mock_news.assert_called_once_with(fake_api_keys["news"])
    mock_weather.assert_called_once_with(fake_api_keys["weather"], "Paris")
```

### Task 3c: Failure path

```python
@pytest.mark.asyncio
async def test_missing_news_api_key_raises(make_weather, fake_api_keys):
    with patch("src.aggregator.fetch_weather", new_callable=AsyncMock) as mock_weather:
        mock_weather.return_value = make_weather(city="Berlin")

        with pytest.raises(APIKeyMissingError, match="NEWS_API_KEY"):
            await build_report(
                news_api_key="",                     # ← triggers the real guard clause
                weather_api_key=fake_api_keys["weather"],
                city="Berlin",
            )
```

**Why `fetch_news` is NOT mocked here:**
The guard `if not api_key: raise APIKeyMissingError(...)` lives inside `fetch_news`. Mocking `fetch_news` would bypass that guard entirely — the mock returns a list regardless of the key. We want to verify the real guard works. `fetch_weather` is mocked to prevent any real HTTP attempt if it were to run before the news task raises and `asyncio.gather` propagates the exception.

### Task 3d: Parametrized city

```python
@pytest.mark.parametrize(
    "city",
    ["London", "Tokyo", "São Paulo"],
    ids=["london", "tokyo", "sao_paulo"],
)
@pytest.mark.asyncio
async def test_weather_fetched_for_correct_city(
    city: str, make_news_item, make_weather, fake_api_keys
):
    with patch("src.aggregator.fetch_news", new_callable=AsyncMock) as mock_news, \
         patch("src.aggregator.fetch_weather", new_callable=AsyncMock) as mock_weather:

        mock_news.return_value    = [make_news_item()]
        mock_weather.return_value = make_weather(city=city)

        await build_report(
            news_api_key=fake_api_keys["news"],
            weather_api_key=fake_api_keys["weather"],
            city=city,
        )

    mock_weather.assert_called_once_with(fake_api_keys["weather"], city)
```

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## Exercise 4 — When Mocking Makes Your Tests Worse

**The Scenario:**

A developer writes "integration tests" for a data access layer. To keep them "fast and isolated," they mock SQLAlchemy's `Session`. All tests pass. Production breaks on the first real database write.

---

**Part A: Name the Category**

What specific category of test should never mock the persistence layer? In one paragraph, explain why replacing a real database session with a `MagicMock` produces false confidence rather than genuine verification.

---

**Part B: The Broken "Integration" Test**

```python
# src/user_repository.py
from sqlalchemy.orm import Session
from src.models import User  # User has a UNIQUE constraint on username

def create_user(db: Session, username: str, email: str) -> User:
    """Create a user. The DB raises IntegrityError if username already exists."""
    user = User(username=username, email=email)
    db.add(user)
    db.commit()
    db.refresh(user)
    return user
```

```python
# tests/test_user_repository.py — FLAWED "integration" test
from unittest.mock import MagicMock
from src.user_repository import create_user

def test_duplicate_username_is_rejected():
    """Claims to verify UNIQUE constraint enforcement."""
    mock_db = MagicMock()

    create_user(mock_db, username="alice", email="alice@example.com")
    create_user(mock_db, username="alice", email="other@example.com")  # duplicate!

    # The test writer is stuck: how do they assert rejection?
    # The mock never raised IntegrityError. Both "commits" succeeded.
    assert mock_db.commit.call_count == 2   # passes — proves nothing
```

Identify precisely what the mock is incapable of doing here, and explain why mocking `Session` is the root cause of the test's inability to detect the duplicate.

---

**Part C: When IS Mocking the Session Correct?**

The same `create_user` function also has legitimate unit tests. Describe a unit test scenario where mocking `db` IS appropriate, write the test, and explain exactly what it verifies that a real database cannot (and need not) verify.

---

## Level 4 Solutions

**Part A:**
**Integration tests** should never mock the persistence layer. The purpose of an integration test is to verify that your application code works correctly *with the real system it will use in production* — in this case, PostgreSQL. Replacing the real database with a `MagicMock` produces a test that verifies only that your code makes the correct method calls on an object that blindly accepts everything. The mock does not know about `UNIQUE` constraints, foreign key enforcement, transaction isolation levels, or the difference between an `INTEGER` column and a `TEXT` column. All of these can cause failures in production that pass perfectly in a mocked test suite.

**Part B:**
The `MagicMock` is incapable of enforcing database constraints. When the real PostgreSQL receives the second `INSERT` with `username="alice"`, it checks the `UNIQUE` index and raises `IntegrityError`, which SQLAlchemy wraps and propagates. `MagicMock.commit()` does nothing — it accepts all calls and returns `MagicMock()`. The mock has no knowledge of what was "inserted" before it, no index to check, and no constraint to enforce. The assertion `assert mock_db.commit.call_count == 2` confirms two commits were made — which is trivially true and proves only that the function calls `commit()`, not that the database rejects duplicates.

**Part C:**
```python
from unittest.mock import MagicMock, call
from src.user_repository import create_user

def test_create_user_calls_session_in_correct_order():
    """
    Unit test: verify that create_user calls add(), commit(), refresh() in order.
    This tests the CALL PATTERN, not the database behaviour.
    A real database cannot be used to test call order — it only exposes outcomes.
    """
    mock_db = MagicMock()

    create_user(mock_db, username="alice", email="alice@example.com")

    assert mock_db.add.call_count == 1
    assert mock_db.commit.call_count == 1
    assert mock_db.refresh.call_count == 1
    # Verify add() was called before commit() by checking call order:
    assert mock_db.method_calls.index(call.add(mock_db.add.call_args[0][0])) < \
           mock_db.method_calls.index(call.commit())
```

This is a legitimate unit test: it verifies that `create_user` uses the session interface correctly — calls `add`, then `commit`, then `refresh` in that order. This is the function's *contract with the session interface*. The database integration test (with a real PostgreSQL test database) verifies the database's *contract with your data*. Both tests are needed. They test different, non-overlapping things.

---

# LEVEL 5: THE TEACH BACK

## Exercise 5 — Correcting the Overconfident Developer

**The Scenario:**

A junior developer presents their test suite at code review:

> "Here is my confidence checklist for the mini-project:
>
> ✅ 97% line coverage  
> ✅ All 24 tests pass  
> ✅ Every `async def` dependency is replaced with `AsyncMock`  
> ✅ `fetch_news` and `fetch_weather` are mocked in every single test  
> ✅ Full suite completes in 0.6 seconds
>
> This is a well-tested codebase. I'm confident nothing will break in production."

They show you three tests:

```python
# Test 1 — "Tests the aggregator"
@pytest.mark.asyncio
async def test_build_report():
    with patch("src.aggregator.build_report", new_callable=AsyncMock) as mock_build:
        mock_build.return_value = AggregatedReport(
            news=[], weather=WeatherSummary(city="London", temp_celsius=10.0),
            fetched_at=datetime.now(),
        )
        result = await build_report("k1", "k2", city="London")
    assert result.weather.city == "London"

# Test 2 — "Validates the news fetcher logic"
@pytest.mark.asyncio
async def test_fetch_news_runs():
    with patch("src.aggregator.fetch_news", new_callable=AsyncMock) as mock_news:
        mock_news.return_value = []
        result = await fetch_news("fake-key")
    assert isinstance(result, list)

# Test 3 — "Tests error handling for missing key"
@pytest.mark.asyncio
async def test_missing_key_error():
    with pytest.raises(Exception):      # catches any exception
        await fetch_news("")            # fetch_news is NOT mocked here
```

**Questions:**

1. **Test 1 — The Self-Mock:** Explain why this test verifies nothing about `build_report`'s actual behaviour. Name one specific defect in `build_report` that this test is structurally incapable of detecting.

2. **Test 2 — The Coverage Lie:** `assert isinstance(result, list)` achieves line coverage on `fetch_news`. What specific bug in `fetch_news`'s parsing logic would this assertion permanently miss? How does it relate to the Coverage Illusion from Exercise 1.3?

3. **Test 3 — The Exception Black Hole:** `pytest.raises(Exception)` catches the error. Why is this dangerous? What does `pytest.raises(APIKeyMissingError, match="NEWS_API_KEY")` add that `pytest.raises(Exception)` cannot provide? Connect your answer to Week 1 Lecture 2's custom exception hierarchy lesson.

4. **The Synthesis:** In 3–4 sentences, explain the relationship between coverage, mock depth, and actual correctness. What is the minimum set of properties that, taken together, constitute a genuinely well-tested codebase?

---

## Level 5 Solutions

**1. Test 1 — The Self-Mock:**
`build_report` itself has been patched with `mock_build`. The call `await build_report("k1", "k2", city="London")` invokes the mock — not the real function. The real `build_report`, which contains `asyncio.create_task`, `asyncio.gather`, and `AggregatedReport(fetched_at=datetime.now())`, is never executed. `result.weather.city == "London"` passes because the mock was told to return exactly that. This test is identical in value to writing `assert "London" == "London"`.

A specific defect it cannot detect: if `build_report` were changed to always pass `city="Paris"` to `fetch_weather` regardless of the `city` parameter, this test would still pass. The mock bypasses the argument-passing logic entirely.

**2. Test 2 — The Coverage Lie:**
`fetch_news` is mocked, so when `await fetch_news("fake-key")` is called, the mock returns `[]` without executing a single line of the real function. Despite 100% coverage being reported (because the mock causes the lines to be "executed" in other tests perhaps, or this test executes the mock path), `assert isinstance([], list)` is trivially true and verifies nothing.

The specific bug permanently hidden: if the list comprehension `[NewsItem(title=a["title"], source=a["source"]["name"]) for a in data.get("articles", [])]` had a typo — `a["sorce"]` instead of `a["source"]` — the `isinstance` check would still pass as long as the mock returned `[]`. The exact same failure mode as Exercise 1.3: coverage ≠ correctness.

**3. Test 3 — The Exception Black Hole:**
`pytest.raises(Exception)` accepts any exception from any cause — `APIKeyMissingError`, but also `httpx.ConnectError`, `RuntimeError`, a `KeyError` from a typo, or even a `SystemExit`. If someone accidentally introduced an unrelated bug in `fetch_news` that causes `TypeError`, this test would pass and silently report "error handling works." The test provides no evidence that the right error, for the right reason, was raised.

`pytest.raises(APIKeyMissingError, match="NEWS_API_KEY")` adds two guarantees that connect directly to Week 1 Lecture 2's custom exception lesson:
- **Type specificity:** Only `APIKeyMissingError` (or a subclass) satisfies the assertion. Every other exception — including bugs — causes the test to fail, surfacing the real problem.
- **Message specificity:** The `match=` regex confirms the message contains `"NEWS_API_KEY"`, proving the exception was raised by the missing-key guard and not some other error path. This is exactly why custom exceptions carry context: without a specific message to assert against, you cannot distinguish *why* the error occurred, only *that* it did.

**4. The Synthesis:**
Coverage measures which lines were executed; it says nothing about whether those lines produced correct output. Mock depth controls what code actually runs: mocking the function under test means zero real code runs and the test proves nothing. A genuinely well-tested codebase requires four properties working together: **(1)** assertions that verify specific computed values — not types or existence; **(2)** mocks applied only at external boundaries (HTTP, time, file system), never to the function being tested; **(3)** exception tests that name the specific custom exception type and validate the message with `match=`; and **(4)** at least one layer of integration tests using real components, because mocks are architecturally incapable of detecting the class of bugs that only real systems can produce.