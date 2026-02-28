# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FEAR FIRST, TOOLS SECOND                                       │
│  ────────────────────────                                       │
│  Students must feel the TERROR of silent breakage before        │
│  they care about writing tests. We break their code live.       │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Testing is abstract until you compare it to something real.    │
│  We use a factory quality control (QC) analogy throughout.      │
│  Every concept maps to an inspection stage on the factory       │
│  floor.                                                         │
│                                                                 │
│  CODE THEY ALREADY KNOW                                         │
│  ──────────────────────                                         │
│  Every example uses dataclasses, type hints, custom             │
│  exceptions, and async — things from Week 1. Testing is not     │
│  a new world. It's a lens on code they've already written.      │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Decorators → @pytest.fixture, @pytest.mark.parametrize         │
│  Context managers → yield fixtures ARE context managers         │
│  Custom exceptions → pytest.raises tests your hierarchies       │
│  Type hints → tests verify what mypy cannot (logic errors)      │
│  Debugging → when a test fails, your Week 2 L1 skills kick in   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE
```
┌─────────────────────────────────────────────────────────────────┐
│                   TESTING FUNDAMENTALS                          │
│                     (Extended Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (40 min)                                   │
│  ├─ 1.1 The Silent Breakage (Demonstration)                     │
│  ├─ 1.2 What Testing Actually Is                                │
│  ├─ 1.3 The Three Gifts of Testing                              │
│  └─ 1.4 The Factory QC Analogy                                  │
│                                                                 │
│  PART 2: PYTEST FUNDAMENTALS (55 min)                           │
│  ├─ 2.1 Your First Test                                         │
│  ├─ 2.2 Assertions — The Inspector's Checklist                  │
│  ├─ 2.3 Testing Exceptions (Connection to Lecture 2, Week 1)    │
│  ├─ 2.4 Parametrize — One Test, Many Cases                      │
│  ├─ 2.5 Running and Reading pytest Output                       │
│  ├─ 2.6 Skipping Tests and Expected Failures                    │
│  ├─ 2.7 Custom Markers and Advanced Parametrize                 │
│  └─ 2.8 Debugging Failing Tests                                 │
│                                                                 │
│  PART 3: FIXTURES (45 min)                                      │
│  ├─ 3.1 The Problem: Repeated Setup                             │
│  ├─ 3.2 @pytest.fixture — The Test Workbench                    │
│  ├─ 3.3 Fixture Scope (function, module, session)               │
│  ├─ 3.4 Yield Fixtures (Connection to Context Managers)         │
│  ├─ 3.5 conftest.py — The Shared Toolroom                       │
│  └─ 3.6 Built-in Fixtures (tmp_path, monkeypatch,               │ 
│          capsys, capfd, caplog)                                 │
│                                                                 │
│  PART 4: THE TESTING PYRAMID (35 min)                           │
│  ├─ 4.1 Three Levels of Inspection                              │
│  ├─ 4.2 Unit Tests (Testing One Gear)                           │
│  ├─ 4.3 Integration Tests (Testing the Assembly)                │
│  ├─ 4.4 End-to-End Tests (Test-Driving the Car)                 │
│  └─ 4.5 The Pyramid Shape (Why More Unit, Fewer E2E)            │
│                                                                 │
│  PART 5: TEST DOUBLES (40 min)                                  │
│  ├─ 5.1 The Isolation Problem                                   │
│  ├─ 5.2 Mocks, Stubs, and Fakes — The Differences               │
│  ├─ 5.3 unittest.mock.patch — Swapping Real for Fake            │
│  ├─ 5.4 MagicMock and AsyncMock                                 │
│  └─ 5.5 When Mocking Goes Wrong                                 │
│                                                                 │
│  PART 6: TEST-DRIVEN DEVELOPMENT (25 min)                       │
│  ├─ 6.1 Red-Green-Refactor — The TDD Cycle                      │
│  ├─ 6.2 TDD Walkthrough (Live Coding)                           │
│  └─ 6.3 When TDD Helps, When It Doesn't                         │
│                                                                 │
│  PART 7: PYTEST CONFIGURATION AND COVERAGE                      │
│  ├─ 7.1 Configuring pytest in pyproject.toml                    │
│  └─ 7.2 Measuring Coverage with pytest-cov                      │
│                                                                 │
│  PART 8: PRACTICAL TESTING PATTERNS                             │
│  ├─ 8.1 Testing CLI Applications                                │
│  ├─ 8.2 Time-Dependent Testing (freezegun)                      │
│  ├─ 8.3 Warnings Testing (pytest.warns)                         │
│  ├─ 8.4 Property-Based Testing (hypothesis — intro)             │
│  ├─ 8.5 Test Data Factories                                     │
│  ├─ 8.6 Test Parallelization (pytest-xdist)                     │
│  ├─ 8.7 Test Reporting                                          │
│  └─ 8.8 CI/CD Integration Preview                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

# PART 1: THE PROBLEM

## 1.1 The Silent Breakage

**Start with a demonstration. Make them feel the fear.**

```python
# demo_before.py — Show this code first. "This works. Trust me."
from dataclasses import dataclass

@dataclass
class Order:
    item: str
    quantity: int
    unit_price: float

def calculate_total(order: Order, discount_percent: float = 0.0) -> float:
    """Calculate the total price after discount."""
    subtotal = order.quantity * order.unit_price
    discount = subtotal * (discount_percent / 100)
    return subtotal - discount

def apply_bulk_discount(order: Order) -> float:
    """Orders of 10+ items get 15% off."""
    if order.quantity >= 10:
        return calculate_total(order, discount_percent=15.0)
    return calculate_total(order)

# Demo: it works
order = Order(item="Widget", quantity=12, unit_price=10.0)
total = apply_bulk_discount(order)
print(f"Total: ${total:.2f}")  # Total: $102.00 ✅ Correct!
```

**Now tell the class: "Your teammate refactored `calculate_total` last night."**

```python
# demo_after.py — The "refactored" version. Spot the bug?
from dataclasses import dataclass

@dataclass
class Order:
    item: str
    quantity: int
    unit_price: float

def calculate_total(order: Order, discount_percent: float = 0.0) -> float:
    """Calculate the total price after discount."""
    subtotal = order.quantity * order.unit_price
    # "Refactored" — Teammate simplified the math.
    return subtotal * (discount_percent / 100)  # 🐛 BUG!
    # Returns the DISCOUNT AMOUNT, not the discounted price!

def apply_bulk_discount(order: Order) -> float:
    """Orders of 10+ items get 15% off."""
    if order.quantity >= 10:
        return calculate_total(order, discount_percent=15.0)
    return calculate_total(order)

# Demo: looks fine at a glance...
order = Order(item="Widget", quantity=12, unit_price=10.0)
total = apply_bulk_discount(order)
print(f"Total: ${total:.2f}")  # Total: $18.00 ← WRONG! Should be $102.00
```

**Run it. No crash. No error. No traceback.**

```
Total: $18.00
```

**Now ask the class:**

> "This code runs perfectly. Python doesn't complain. mypy doesn't complain — the types are all correct. Your teammate's PR got merged at 11 PM. Your customers are now being charged $18 instead of $102. How long until someone notices?"

**Let that sink in.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE TIMELINE OF PAIN                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  11:00 PM — Teammate merges "refactored" code                   │
│  11:01 PM — Deployed to production                              │
│  11:02 PM — First customer charged $18 instead of $102          │
│  11:03 PM — Second customer. Third. Fourth.                     │
│  ...                                                            │
│  ...       Nobody notices. Code runs fine. No errors.           │
│  ...                                                            │
│   8:00 AM — Finance team: "Why is revenue down 82%?"            │
│   8:30 AM — 9 hours of wrong charges. Thousands of dollars.     │
│   9:00 AM — Your boss: "How did this pass code review?"         │
│                                                                 │
│  ❓ What could have caught this at 11:00 PM, BEFORE deploy?     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The answer:**

```python
# This single line would have caught the bug INSTANTLY:
def test_calculate_total_with_discount():
    order = Order(item="Widget", quantity=12, unit_price=10.0)
    assert calculate_total(order, discount_percent=15.0) == 102.0
```

```
FAILED test_orders.py::test_calculate_total_with_discount
  assert 18.0 == 102.0
```

> "One line. One assertion. $0 in lost revenue. That's what testing is."

---

## 1.2 What Testing Actually Is

**Clear up the misconception first:**

```
┌─────────────────────────────────────────────────────────────────┐
│                TESTING IS NOT "RUNNING YOUR CODE"               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ WHAT TESTING IS NOT:                                        │
│  ├─ "I ran it and it printed the right thing"                   │
│  ├─ "I tried a few inputs manually in the terminal"             │
│  ├─ "It didn't crash, so it works"                              │
│  └─ "I'll test it later when I have time"                       │
│                                                                 │
│  ✅ WHAT TESTING IS:                                            │
│  ├─ Code that verifies other code — AUTOMATICALLY               │
│  ├─ Runs the SAME checks every time, no human needed            │
│  ├─ Catches regressions BEFORE they reach production            │
│  └─ Runs in seconds, every time you push code                   │
│                                                                 │
│                                                                 │
│  MANUAL CHECKING:                                               │
│     You → run code → look at output → "looks right" → 🤞        │
│                                                                 │
│  AUTOMATED TESTING:                                             │
│     pytest → runs 200 checks → all pass → ✅ VERIFIED           │
│     pytest → runs 200 checks → 1 fails → ❌ BUG FOUND           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connect to Debugging (Week 2, Lecture 1):**

> "Last lecture you learned how to find bugs AFTER they appear. Testing is the opposite — it's how you prevent bugs from surviving past your editor. Debugging is the fire department. Testing is the fire alarm."

---

## 1.3 The Three Gifts of Testing

```
┌─────────────────────────────────────────────────────────────────┐
│                THE THREE GIFTS OF TESTING                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  🎁 GIFT 1: CONFIDENCE                                          │
│  ──────────────────────                                         │
│  "Does my code actually work?"                                  │
│  Without tests: "I think so... I ran it once..."                │
│  With tests: "Yes. 47 checks pass. Here's the proof."           │
│                                                                 │
│  You deploy on Friday afternoon without sweating.               │
│                                                                 │
│                                                                 │
│  🎁 GIFT 2: REFACTORING SAFETY                                  │
│  ────────────────────────────                                   │
│  "Can I change this code without breaking something?"           │
│  Without tests: "I'm afraid to touch it."                       │
│  With tests: "I'll change it and run the tests. If they         │
│   pass, I haven't broken anything."                             │
│                                                                 │
│  Tests are the reason you CAN improve old code.                 │
│                                                                 │
│                                                                 │
│  🎁 GIFT 3: DOCUMENTATION                                       │
│  ─────────────────────────                                      │
│  "What is this function supposed to do?"                        │
│  Without tests: Read the code, guess the intent.                │
│  With tests: Read the tests — they show EXACTLY                 │
│   what inputs produce what outputs.                             │
│                                                                 │
│  Tests are executable documentation that never goes stale.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Gift 3 illustrated:**

```python
# Which is easier to understand?

# OPTION A: Read the function
def apply_bulk_discount(order: Order) -> float:
    if order.quantity >= 10:
        return calculate_total(order, discount_percent=15.0)
    return calculate_total(order)
# "OK... so if quantity >= 10... 15%... I think I get it?"

# OPTION B: Read the tests
def test_bulk_discount_applied_for_10_or_more():
    order = Order(item="Widget", quantity=10, unit_price=10.0)
    assert apply_bulk_discount(order) == 85.0  # 15% off $100

def test_no_discount_for_fewer_than_10():
    order = Order(item="Widget", quantity=9, unit_price=10.0)
    assert apply_bulk_discount(order) == 90.0  # Full price

def test_bulk_discount_applies_to_large_orders():
    order = Order(item="Widget", quantity=100, unit_price=5.0)
    assert apply_bulk_discount(order) == 425.0  # 15% off $500
# Crystal clear. No guessing.
```

**Connect to Type Hints (Week 1, Lecture 1):**

> "Remember type hints? mypy catches *type* errors — passing a string where an int is expected. But mypy can't catch *logic* errors — returning 18.0 instead of 102.0. The types were correct! Testing catches what mypy cannot. They're partners, not replacements."

```
┌─────────────────────────────────────────────────────────────────┐
│                YOUR SAFETY NET STACK                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Layer 1: Type Hints + mypy                                     │
│           Catches: wrong types, missing attributes              │
│           Misses:  logic errors, wrong calculations             │
│                                                                 │
│  Layer 2: Automated Tests (pytest)                              │
│           Catches: wrong behavior, logic errors, regressions    │
│           Misses:  issues only visible in real environment      │
│                                                                 │
│  Layer 3: Debugging (pdb, breakpoint)                           │
│           Catches: whatever slipped through Layers 1 & 2        │
│           Used:    when tests fail and you need to find WHY     │
│                                                                 │
│  Each layer catches what the layer above misses.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 The Factory QC Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE FACTORY QC ANALOGY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOUR CODE IS A CAR FACTORY.                                    │
│  Your functions are the machines building parts.                │
│  Your users are the customers buying the car.                   │
│                                                                 │
│  FACTORY WITHOUT QUALITY CONTROL:                               │
│  ────────────────────────────────                               │
│                                                                 │
│  1. Build engine           ──┐                                  │
│  2. Build transmission     ──┤                                  │
│  3. Bolt them together     ──┤── "Ship it!"                     │
│  4. Build chassis          ──┤                                  │
│  5. Assemble car           ──┘                                  │
│  6. Customer drives off a cliff.  💥                            │
│                                                                 │
│                                                                 │
│  FACTORY WITH QUALITY CONTROL:                                  │
│  ─────────────────────────────                                  │
│                                                                 │
│  1. Build engine           → Test engine alone ✅               │
│  2. Build transmission     → Test transmission alone ✅         │
│  3. Bolt them together     → Test engine+transmission ✅        │
│  4. Build chassis          → Test chassis alone ✅              │
│  5. Assemble car           → Test-drive the whole car ✅        │
│  6. Customer drives home safely. 🎉                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to testing:**

```
Factory QC             │  Python Testing
───────────────────────│─────────────────────────────
Individual part test   │  Unit test
Assembly test          │  Integration test
Full car test-drive    │  End-to-end (E2E) test
QC test bench (tools,  │  Fixture (test setup)
  jigs, calibrators)   │    
Dummy engine for       │  Mock / Stub (fake object
  testing transmission │    replacing a real dependency)
QC checklist item      │  Assertion (assert x == y)
Spec sheet written     │  TDD (write tests before code)
  before manufacturing │
QC report              │  pytest output
```

---

# PART 2: PYTEST FUNDAMENTALS

## 2.1 Your First Test

**pytest uses plain Python functions. No classes needed. No boilerplate.**

```python
# src/orders.py — The code we want to test
from dataclasses import dataclass

@dataclass
class Order:
    item: str
    quantity: int
    unit_price: float

def calculate_total(order: Order, discount_percent: float = 0.0) -> float:
    """Calculate total price after discount."""
    subtotal = order.quantity * order.unit_price
    discount = subtotal * (discount_percent / 100)
    return subtotal - discount
```

```python
# tests/test_orders.py — Your first test file
from src.orders import Order, calculate_total

def test_calculate_total_no_discount():
    order = Order(item="Widget", quantity=5, unit_price=10.0)
    result = calculate_total(order)
    assert result == 50.0
```

**That's it. Three lines of actual code. Let's break down the rules:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PYTEST DISCOVERY RULES                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  pytest AUTOMATICALLY finds your tests. It looks for:           │
│                                                                 │
│  FILES:                                                         │
│  ├─ Files named test_*.py     ✅ test_orders.py                 │
│  ├─ Files named *_test.py     ✅ orders_test.py                 │
│  └─ Other names               ❌ my_checks.py (not found)       │
│                                                                 │
│  FUNCTIONS inside those files:                                  │
│  ├─ Functions named test_*    ✅ def test_calculate_total():    │
│  └─ Other names               ❌ def check_total(): (skipped)   │
│                                                                 │
│  FOLDERS:                                                       │
│  └─ pytest searches recursively from where you run it           │
│     Usually: project_root/tests/                                │
│                                                                 │
│  CONVENTION — you don't configure any of this.                  │
│  Just name things right, and pytest finds them.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Running it:**

```bash
$ pytest tests/test_orders.py -v

========================= test session starts ==========================
collected 1 item

tests/test_orders.py::test_calculate_total_no_discount PASSED      [100%]

========================== 1 passed in 0.01s ===========================
```

**The anatomy of a test function:**

```python
def test_calculate_total_no_discount():        # ← NAME: says what it tests
    # ARRANGE — set up the inputs
    order = Order(item="Widget", quantity=5, unit_price=10.0)
    
    # ACT — call the function being tested
    result = calculate_total(order)
    
    # ASSERT — verify the output
    assert result == 50.0
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE AAA PATTERN                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Every test follows three steps:                                │
│                                                                 │
│  ARRANGE  → Set up the data, objects, and conditions            │
│  ACT      → Call the function/method you're testing             │
│  ASSERT   → Verify the result is what you expected              │
│                                                                 │
│  Factory analogy:                                               │
│  ARRANGE  → Place the part on the test bench                    │
│  ACT      → Run the machine / apply pressure                    │
│  ASSERT   → Check: did it pass the specification?               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Test naming matters — your test name IS your documentation:**

```python
# ❌ BAD: What does this test?
def test_1():
    ...

# ❌ BAD: Too vague
def test_total():
    ...

# ✅ GOOD: Reads like a sentence
def test_calculate_total_with_no_discount_returns_full_price():
    ...

# ✅ GOOD: Describes the scenario
def test_calculate_total_with_20_percent_discount():
    ...

# ✅ GOOD: Describes edge case
def test_calculate_total_with_zero_quantity_returns_zero():
    ...
```

---

## 2.2 Assertions — The Inspector's Checklist

**`assert` is Python's built-in keyword. pytest makes it powerful.**

```python
# Basic assertions — you already know these
def test_equality():
    assert 1 + 1 == 2

def test_truthiness():
    assert "hello"          # Non-empty string is truthy
    assert not ""           # Empty string is falsy
    assert not None         # None is falsy

def test_membership():
    assert "a" in ["a", "b", "c"]
    assert "d" not in ["a", "b", "c"]

def test_identity():
    assert None is None
    assert calculate_total(Order("x", 0, 0)) is not None

def test_comparison():
    assert 10 > 5
    assert 3.14 < 4.0
```

**pytest's secret weapon — assertion introspection:**

When a regular `assert` fails, Python says almost nothing:

```python
# Plain Python:
assert 18.0 == 102.0
# AssertionError  ← Thanks for nothing, Python.
```

But pytest *rewrites* your assertions to give detailed diagnostics:

```python
# pytest output:
def test_calculate_total_with_discount():
    order = Order(item="Widget", quantity=12, unit_price=10.0)
>   assert calculate_total(order, discount_percent=15.0) == 102.0
E   assert 18.0 == 102.0
E    +  where 18.0 = calculate_total(Order(item='Widget', quantity=12, unit_price=10.0), discount_percent=15.0)
```

```
┌─────────────────────────────────────────────────────────────────┐
│              PYTEST ASSERTION INTROSPECTION                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  pytest rewrites your assert statements at import time          │
│  to capture intermediate values. You get:                       │
│                                                                 │
│  • The EXACT values on both sides of ==                         │
│  • The function call that produced them                         │
│  • For collections: a DIFF showing what's different             │
│                                                                 │
│  You DON'T need:                                                │
│  ├─ assertEqual()    ← unittest style, not needed               │
│  ├─ assertTrue()     ← just use assert                          │
│  └─ assertIn()       ← just use assert x in y                   │
│                                                                 │
│  Plain assert is all you need. pytest handles the rest.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Comparing complex objects — pytest shows diffs:**

```python
def test_order_data():
    result = {
        "item": "Widget",
        "quantity": 12,
        "price": 18.0,       # ← Wrong!
        "status": "pending"
    }
    expected = {
        "item": "Widget",
        "quantity": 12,
        "price": 102.0,      # ← Expected
        "status": "pending"
    }
    assert result == expected
```

```
E   AssertionError: assert {'item': 'Wid...tus': 'pending'} == {'item': 'Wid...tus': 'pending'}
E     Differing items:
E     {'price': 18.0} != {'price': 102.0}
```

**Floating point comparison — a common trap:**

```python
# ❌ DANGEROUS: Floating point arithmetic is not exact
def test_float_equality():
    assert 0.1 + 0.2 == 0.3  # FAILS! 0.30000000000000004 != 0.3

# ✅ CORRECT: Use pytest.approx for float comparison
import pytest

def test_float_with_approx():
    assert 0.1 + 0.2 == pytest.approx(0.3)  # PASSES!

# You can set tolerance:
def test_float_with_tolerance():
    assert 3.14 == pytest.approx(3.14159, abs=0.01)  # Within 0.01
```

---

## 2.3 Testing Exceptions (Connection to Lecture 2, Week 1)

**Connection to what you've learned:**

> "Remember custom exception hierarchies from Week 1, Lecture 2? You built `APIError`, `NotFoundError`, `RateLimitError`. Now you'll learn how to TEST that your code raises the right exceptions."

```python
# src/orders.py — Using custom exceptions (Week 1 Lecture 2 pattern)

class OrderError(Exception):
    """Base error for order operations."""
    pass

class InvalidQuantityError(OrderError):
    """Raised when quantity is invalid."""
    def __init__(self, quantity: int):
        self.quantity = quantity
        super().__init__(f"Invalid quantity: {quantity}. Must be positive.")

class InvalidDiscountError(OrderError):
    """Raised when discount is out of range."""
    def __init__(self, discount: float):
        self.discount = discount
        super().__init__(f"Invalid discount: {discount}%. Must be 0-100.")

def calculate_total(order: Order, discount_percent: float = 0.0) -> float:
    if order.quantity <= 0:
        raise InvalidQuantityError(order.quantity)
    if not (0 <= discount_percent <= 100):
        raise InvalidDiscountError(discount_percent)
    
    subtotal = order.quantity * order.unit_price
    discount = subtotal * (discount_percent / 100)
    return subtotal - discount
```

**`pytest.raises` — assert that an exception IS raised:**

```python
import pytest
from src.orders import (
    Order, calculate_total,
    InvalidQuantityError, InvalidDiscountError, OrderError
)

# Basic: just check the exception type
def test_negative_quantity_raises_error():
    order = Order(item="Widget", quantity=-1, unit_price=10.0)
    with pytest.raises(InvalidQuantityError):
        calculate_total(order)

# Advanced: inspect the exception object
def test_negative_quantity_error_has_details():
    order = Order(item="Widget", quantity=-5, unit_price=10.0)
    with pytest.raises(InvalidQuantityError) as exc_info:
        calculate_total(order)
    
    assert exc_info.value.quantity == -5
    assert "Must be positive" in str(exc_info.value)

# Test the exception hierarchy (Week 1 Lecture 2 payoff!)
def test_invalid_quantity_is_an_order_error():
    order = Order(item="Widget", quantity=0, unit_price=10.0)
    with pytest.raises(OrderError):  # Parent class works too!
        calculate_total(order)
```

**`pytest.raises` uses the context manager pattern (Week 1, Lecture 2 callback):**

```
┌─────────────────────────────────────────────────────────────────┐
│               pytest.raises IS A CONTEXT MANAGER                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Remember the `with` statement from Week 1?                     │
│                                                                 │
│  with open("file.txt") as f:                                    │
│      data = f.read()                                            │
│  # File automatically closed                                    │
│                                                                 │
│  Same pattern:                                                  │
│                                                                 │
│  with pytest.raises(SomeError) as exc_info:                     │
│      code_that_should_raise()                                   │
│  # pytest checks: was SomeError raised?                         │
│  # If yes: test passes, exception captured in exc_info          │
│  # If no:  test FAILS — "SomeError not raised"                  │
│  # If different error: test FAILS — wrong exception             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Match exception messages with `match`:**

```python
def test_discount_error_message():
    order = Order(item="Widget", quantity=5, unit_price=10.0)
    with pytest.raises(InvalidDiscountError, match=r"Invalid discount: 150"):
        calculate_total(order, discount_percent=150.0)
    # `match` accepts a regex pattern checked against str(exception)
```

**Common mistake — testing that NO exception is raised:**

```python
# ❌ WRONG: Don't do this. It's pointless.
def test_valid_order_no_error():
    try:
        order = Order(item="Widget", quantity=5, unit_price=10.0)
        calculate_total(order)
    except Exception:
        pytest.fail("Unexpected exception!")

# ✅ RIGHT: Just call the function. If it raises, pytest catches it.
def test_valid_order_returns_correct_total():
    order = Order(item="Widget", quantity=5, unit_price=10.0)
    assert calculate_total(order) == 50.0
    # If this raises an unexpected exception, pytest reports it as ERROR
```

---

## 2.4 Parametrize — One Test, Many Cases

**The problem: repetitive test code.**

```python
# ❌ Writing the same test structure over and over:
def test_discount_0_percent():
    order = Order(item="W", quantity=10, unit_price=10.0)
    assert calculate_total(order, 0) == 100.0

def test_discount_10_percent():
    order = Order(item="W", quantity=10, unit_price=10.0)
    assert calculate_total(order, 10) == 90.0

def test_discount_50_percent():
    order = Order(item="W", quantity=10, unit_price=10.0)
    assert calculate_total(order, 50) == 50.0

def test_discount_100_percent():
    order = Order(item="W", quantity=10, unit_price=10.0)
    assert calculate_total(order, 100) == 0.0
# Four tests, but 90% of the code is identical! 
```

**The solution: `@pytest.mark.parametrize`**

```python
# ✅ One test function, many input/output pairs:
import pytest

@pytest.mark.parametrize(
    "discount, expected",        # ← Parameter names (comma-separated string)
    [                            # ← List of (input, output) tuples
        (0, 100.0),             # No discount
        (10, 90.0),             # 10% off
        (25, 75.0),             # 25% off
        (50, 50.0),             # Half off
        (100, 0.0),             # Free!
    ],
)
def test_calculate_total_with_various_discounts(discount: float, expected: float):
    order = Order(item="Widget", quantity=10, unit_price=10.0)
    assert calculate_total(order, discount) == expected
```

**Connection to Decorators (Week 1, Lecture 2):**

> "Notice the `@` syntax? This is a decorator — exactly what you learned in Week 1. `@pytest.mark.parametrize` transforms one test function into MANY test cases. pytest runs each tuple as a separate test."

**Running parametrized tests:**

```bash
$ pytest tests/test_orders.py -v

tests/test_orders.py::test_calculate_total_with_various_discounts[0-100.0]    PASSED
tests/test_orders.py::test_calculate_total_with_various_discounts[10-90.0]    PASSED
tests/test_orders.py::test_calculate_total_with_various_discounts[25-75.0]    PASSED
tests/test_orders.py::test_calculate_total_with_various_discounts[50-50.0]    PASSED
tests/test_orders.py::test_calculate_total_with_various_discounts[100-0.0]    PASSED

========================== 5 passed in 0.02s ===========================
```

**Parametrize with multiple parameters:**

```python
@pytest.mark.parametrize(
    "quantity, unit_price, discount, expected",
    [
        (1,  10.0,  0,   10.0),    # Single item, no discount
        (5,  10.0,  0,   50.0),    # Multiple items, no discount
        (5,  10.0,  20,  40.0),    # Multiple items, 20% off
        (1,  99.99, 10,  89.991),  # Decimal price, 10% off
    ],
)
def test_calculate_total_comprehensive(
    quantity: int, unit_price: float, discount: float, expected: float
):
    order = Order(item="Widget", quantity=quantity, unit_price=unit_price)
    assert calculate_total(order, discount) == pytest.approx(expected)
```

**Parametrize for error cases too:**

```python
@pytest.mark.parametrize(
    "quantity, expected_error",
    [
        (0,  InvalidQuantityError),
        (-1, InvalidQuantityError),
        (-100, InvalidQuantityError),
    ],
)
def test_invalid_quantities_raise_errors(quantity: int, expected_error: type):
    order = Order(item="Widget", quantity=quantity, unit_price=10.0)
    with pytest.raises(expected_error):
        calculate_total(order)
```

```
┌─────────────────────────────────────────────────────────────────┐
│            WHEN TO PARAMETRIZE VS SEPARATE TESTS                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE PARAMETRIZE when:                                          │
│  ├─ Same logic, different data                                  │
│  ├─ Testing boundary values (0, 1, max, min)                    │
│  └─ The test structure is IDENTICAL — only values change        │
│                                                                 │
│  USE SEPARATE TESTS when:                                       │
│  ├─ Different setup is needed for each case                     │
│  ├─ Different assertions for each case                          │
│  └─ The scenarios are conceptually different                    │
│                                                                 │
│  Rule of thumb: If you can describe all cases as a TABLE        │
│  of inputs → outputs, parametrize. If not, separate tests.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.5 Running and Reading pytest Output

**The CLI options you'll use daily:**

```bash
# Run all tests
$ pytest

# Run with verbose output (see each test name)
$ pytest -v

# Run a specific file
$ pytest tests/test_orders.py

# Run a specific test function
$ pytest tests/test_orders.py::test_calculate_total_no_discount

# Stop on first failure (don't run remaining tests)
$ pytest -x

# Run tests matching a keyword
$ pytest -k "discount"       # Runs all tests with "discount" in the name
$ pytest -k "not slow"       # Skip tests with "slow" in the name

# Show print() output (normally captured)
$ pytest -s

# Show short traceback (less noise)
$ pytest --tb=short

# Show only failures (no passes)
$ pytest --tb=line
```

**Reading a failure — connect to Debugging (Week 2, Lecture 1):**

```
========================= FAILURES ==========================
____________ test_calculate_total_with_discount ______________

    def test_calculate_total_with_discount():
        order = Order(item="Widget", quantity=12, unit_price=10.0)
>       assert calculate_total(order, discount_percent=15.0) == 102.0
E       assert 18.0 == 102.0
E        +  where 18.0 = calculate_total(
E                            Order(item='Widget', quantity=12, unit_price=10.0),
E                            discount_percent=15.0
E                        )

tests/test_orders.py:15: AssertionError
======================== short test summary info ==============
FAILED tests/test_orders.py::test_calculate_total_with_discount
=================== 1 failed, 4 passed in 0.03s ==============
```

```
┌─────────────────────────────────────────────────────────────────┐
│                READING PYTEST FAILURE OUTPUT                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT TO READ (in order):                                       │
│                                                                 │
│  1. BOTTOM LINE — "1 failed, 4 passed"                          │
│     How many tests broke? Start here.                           │
│                                                                 │
│  2. TEST NAME — test_calculate_total_with_discount              │
│     Which specific test failed?                                 │
│                                                                 │
│  3. THE > LINE — shows the exact line that failed               │
│     >   assert calculate_total(order, ...) == 102.0             │
│                                                                 │
│  4. THE E LINES — show the actual vs expected values            │
│     E   assert 18.0 == 102.0                                    │
│     "I got 18.0 but expected 102.0"                             │
│                                                                 │
│  5. FILE:LINE — tests/test_orders.py:15                         │
│     Exact file and line number. Jump straight there.            │
│                                                                 │
│  NOW use your debugging skills from last lecture!               │
│  You know the input. You know the wrong output.                 │
│  Hypothesis → investigate → fix.                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
## 2.6 Skipping Tests and Expected Failures

**Not every test should run in every situation. pytest gives you three tools to handle this honestly.**

```
┌─────────────────────────────────────────────────────────────────┐
│              SKIP vs XFAIL — KNOW THE DIFFERENCE                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @pytest.mark.skip                                              │
│  "Do NOT run this test at all."                                 │
│  Use when: wrong OS, missing dependency, code not written yet.  │
│                                                                 │
│  @pytest.mark.skipif(condition, reason=...)                     │
│  "Skip ONLY IF this condition is true at collection time."      │
│  Use when: platform check, Python version check.                │
│                                                                 │
│  @pytest.mark.xfail                                             │
│  "Run this test, but EXPECT it to fail."                        │
│  Use when: known unresolved bug, test for a future feature.     │
│                                                                 │
│  CRITICAL DIFFERENCE:                                           │
│  skip → code never executes.                                    │
│  xfail → code EXECUTES. Failure is caught gracefully.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
import sys
import pytest
from src.orders import Order, calculate_total, InvalidQuantityError


# ─── @pytest.mark.skip ────────────────────────────────────────────

@pytest.mark.skip(reason="Refund flow not implemented until Sprint 4")
def test_refund_reduces_total():
    """Skipped: this feature does not exist yet."""
    ...
# Output: SKIPPED [100%]


# ─── @pytest.mark.skipif ─────────────────────────────────────────

@pytest.mark.skipif(
    sys.platform == "win32",
    reason="File path separator behaviour differs on Windows"
)
def test_export_uses_forward_slashes():
    ...


# ─── Skipping at runtime (inside the test body) ──────────────────

def test_live_api_call():
    import os
    if not os.environ.get("WEATHER_API_KEY"):
        pytest.skip("WEATHER_API_KEY not set — run manually with real key")
    # ... test that actually hits the API


# ─── @pytest.mark.xfail ──────────────────────────────────────────

@pytest.mark.xfail(reason="Issue #18: floating-point precision not yet handled")
def test_small_quantity_precision():
    order = Order(item="Screw", quantity=3, unit_price=0.10)
    # Bug: returns 0.30000000000000004 instead of 0.30
    assert calculate_total(order) == 0.30
# Output: XFAIL — test suite is still GREEN. The failure is documented.


# ─── xfail with strict=True ──────────────────────────────────────

@pytest.mark.xfail(
    reason="Issue #18: floating-point precision",
    strict=True,        # ← THIS is the important part
)
def test_small_quantity_precision_strict():
    order = Order(item="Screw", quantity=3, unit_price=0.10)
    assert calculate_total(order) == 0.30
# If the bug is fixed and this test NOW passes:
#   Without strict: XPASS — silently allowed (dangerous!)
#   With strict=True: XPASS → FAILED — forces you to remove the mark
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   OUTPUT STATUS CODES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PASSED  — Test ran. All assertions passed.                     │
│  FAILED  — Test ran. An assertion failed or exception raised.   │
│  ERROR   — Test could not even be set up (fixture error, etc.)  │
│  SKIPPED — Test was skipped (skip / skipif / pytest.skip())     │
│  XFAIL   — Expected failure: ran and failed as anticipated.     │
│  XPASS   — Unexpected pass: an xfail test passed.               │
│            (With strict=True, XPASS → FAILED)                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│              SKIP vs XFAIL — DECISION GUIDE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Use SKIP when:                                                 │
│  ├─ Test requires a dependency not installed in this env        │
│  ├─ Test is only valid on a specific OS or Python version       │
│  └─ Feature literally does not exist yet (nothing to run)       │
│                                                                 │
│  Use XFAIL when:                                                │
│  ├─ There is a tracked bug — failure is documented, not hidden  │
│  ├─ Writing a test BEFORE writing the code (TDD Red phase)      │
│  └─ Add strict=True to force removal once the bug is fixed      │
│                                                                 │
│  ⚠️  ANTI-PATTERN: Using skip to hide unexplained failures.     │
│  If you don't know why a test fails → investigate it.           │
│  Skip is not a debug toggle. It is not a mute button.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.7 Custom Markers and Advanced Parametrize

### Custom Markers — Organising Tests with Tags

**The problem:** As your test suite grows past 20 tests, you need to run subsets — run only fast unit tests in pre-commit, run integration tests only in CI, skip slow tests during development.

Custom markers are your tagging system.

```python
# Declare markers in pyproject.toml first (covered in Part 7).
# Here's how to USE them in your test code:

import pytest

@pytest.mark.slow
def test_process_ten_thousand_orders():
    """Batch processing — takes ~8 seconds."""
    orders = [Order(item=f"item_{i}", quantity=i + 1, unit_price=1.0)
              for i in range(10_000)]
    totals = [calculate_total(o) for o in orders]
    assert len(totals) == 10_000


@pytest.mark.integration
def test_order_written_to_file(tmp_path):
    """Requires real file system — not a pure unit test."""
    ...


@pytest.mark.smoke
def test_calculate_total_returns_a_number():
    """Critical path. Always runs, even in the fastest CI check."""
    order = Order(item="Widget", quantity=1, unit_price=10.0)
    assert isinstance(calculate_total(order), float)
```

**Filtering tests by marker:**

```bash
# Run ONLY tests marked as slow:
$ pytest -m slow

# Run EVERYTHING except slow (normal local run):
$ pytest -m "not slow"

# Run smoke OR integration tests:
$ pytest -m "smoke or integration"

# Run integration tests that are not slow:
$ pytest -m "integration and not slow"

# Run ALL tests (no filter):
$ pytest
```

```
┌─────────────────────────────────────────────────────────────────┐
│              COMMON MARKER CONVENTIONS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @pytest.mark.slow        — Tests taking > 1 second             │
│  @pytest.mark.integration — Tests touching real file/DB/network │
│  @pytest.mark.smoke       — Critical path, always run           │
│  @pytest.mark.unit        — Pure logic, no external deps        │
│  @pytest.mark.wip         — Work-in-progress, don't block CI    │
│                                                                 │
│  These are YOUR CONVENTIONS — not pytest built-ins.             │
│  They must be registered in pyproject.toml or you will get:     │
│  PytestUnknownMarkWarning: Unknown pytest.mark.slow             │
│  (Registration covered in Part 7.)                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Advanced Parametrize — `pytest.param` and `ids`

**`pytest.param` — apply marks to individual parameter cases:**

```python
@pytest.mark.parametrize(
    "discount, expected",
    [
        (0,   100.0),    # Normal case
        (10,   90.0),    # Normal case
        (50,   50.0),    # Normal case
        pytest.param(
            150, 0.0,
            marks=pytest.mark.xfail(reason="Issue #22: discount > 100 not validated yet")
        ),
        pytest.param(
            25,  75.0,
            marks=pytest.mark.skip(reason="25% tier removed in pricing v3")
        ),
    ],
)
def test_discount_cases(discount: float, expected: float):
    order = Order(item="Widget", quantity=10, unit_price=10.0)
    assert calculate_total(order, discount) == expected
# 3 cases PASS, 1 is XFAIL, 1 is SKIPPED — independently.
# You do NOT skip or xfail the entire parametrized test.
```

**`ids` — readable test names:**

```bash
# Without ids=, pytest generates this:
tests/test_orders.py::test_totals[5-10.0-0-50.0]       PASSED  ← What is 5-10.0-0-50.0?
tests/test_orders.py::test_totals[10-10.0-15-85.0]     PASSED

# With ids=, pytest generates this:
tests/test_orders.py::test_totals[single_item_no_discount]   PASSED  ← Readable
tests/test_orders.py::test_totals[bulk_order_15pct_off]      PASSED
```

```python
@pytest.mark.parametrize(
    "quantity, unit_price, discount, expected",
    [
        (1,  10.0,  0,   10.0),
        (10, 10.0, 15,   85.0),
        (1,  99.99, 10,  89.991),
    ],
    ids=[
        "single_item_no_discount",
        "bulk_order_15pct_off",
        "decimal_price_10pct_off",
    ]
)
def test_calculate_total_comprehensive(
    quantity: int, unit_price: float, discount: float, expected: float
):
    order = Order(item="Widget", quantity=quantity, unit_price=unit_price)
    assert calculate_total(order, discount) == pytest.approx(expected)
```

---

## 2.8 Debugging Failing Tests

**Connection to Week 2, Lecture 1:**

> "Last lecture, you learned breakpoint(), pdb commands, and the IDE debugger. Those tools do not stop working inside tests. A failing test is your entry point into that same debugging workflow. Here's how to drop directly into it."

**`pytest --pdb` — enter the debugger at the point of failure:**

```bash
$ pytest --pdb tests/test_orders.py::test_calculate_total_with_discount

# When the assertion fails, instead of reporting and exiting,
# pytest DROPS YOU INTO pdb at the exact failing line:

FAILED tests/test_orders.py::test_calculate_total_with_discount
>>>>>>>>>>>>>>>>>>>>>>>> entering PDB >>>>>>>>>>>>>>>>>>>>>>>>>

> tests/test_orders.py(12)test_calculate_total_with_discount()
-> assert calculate_total(order, discount_percent=15.0) == 102.0

(Pdb) pp order                # Inspect the input
Order(item='Widget', quantity=12, unit_price=10.0)

(Pdb) pp calculate_total(order, 15.0)  # Call the function manually
18.0

(Pdb) pp order.quantity * order.unit_price   # Dig into the math
120.0

# From here: use your Week 2 L1 skills. Hypothesis → investigate → fix.
```

**`pytest --trace` — enter the debugger before the test starts:**

```bash
$ pytest --trace tests/test_orders.py::test_calculate_total_with_discount

# Drops into pdb at the FIRST LINE of the test — before anything runs.
# Use when you don't know WHERE in the test the problem is.
# Walk forward with: n (next line), s (step into), c (continue to failure)
```

**`breakpoint()` inside a test — surgical targeting:**

```python
def test_calculate_total_with_discount():
    order = Order(item="Widget", quantity=12, unit_price=10.0)
    result = calculate_total(order, discount_percent=15.0)

    breakpoint()    # ← Drops into pdb HERE, before the assert
    # Inspect: pp result, pp order
    # Run calculate_total manually with different inputs

    assert result == 102.0
```

```bash
# REQUIRED when using breakpoint() — shows the pdb prompt:
$ pytest -s tests/test_orders.py::test_calculate_total_with_discount
# Without -s, pytest captures stdin/stdout and pdb cannot take input.
```

**Useful CLI flags for the debugging workflow:**

```bash
# Show ALL local variables at the failure point (no debugger needed):
$ pytest -l
# or
$ pytest --showlocals

# Output (extra context, no interactive prompt):
# FAILED tests/test_orders.py::test_calculate_total_with_discount
#   order   = Order(item='Widget', quantity=12, unit_price=10.0)
#   result  = 18.0         ← You see this without entering pdb


# Re-run ONLY the tests that failed in the last run:
$ pytest --lf        # lf = "last failed"

# Run failed tests FIRST, then the rest:
$ pytest --ff        # ff = "failed first"
# Useful when you're in a fix-test-fix cycle and want fast feedback.
```

```
┌─────────────────────────────────────────────────────────────────┐
│               TEST DEBUGGING DECISION TREE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Test fails                                                     │
│       │                                                         │
│       ├─ pytest output shows enough context?                    │
│       │   └─ YES → fix directly                                 │
│       │                                                         │
│       ├─ Need to see local variable values?                     │
│       │   └─ pytest -l  (no interactive prompt)                 │
│       │                                                         │
│       ├─ Need to inspect and experiment interactively?          │
│       │   └─ pytest --pdb  (drops in at failure point)          │
│       │                                                         │
│       ├─ Need to walk through the test step by step?            │
│       │   └─ pytest --trace  (starts at line 1)                 │
│       │                                                         │
│       └─ Need to break at a SPECIFIC line mid-test?             │
│           └─ breakpoint() in test body + pytest -s              │
│                                                                 │
│  All pdb commands from Week 2 L1 apply: n, s, c, pp, where     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│         ✏️  PRACTICE CHECKPOINT — PART 2 (2.6 – 2.8)           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. You have a test for a feature that only runs on Linux.      │
│     Write the decorator. Why is skip correct and xfail wrong?   │
│                                                                 │
│  2. A known bug (#42) causes exactly ONE case in a parametrized │
│     test to fail. The other 3 cases are correct. How do you     │
│     mark ONLY that one case as xfail?                           │
│                                                                 │
│  3. An xfail test unexpectedly PASSES after a colleague's PR.   │
│     Without strict=True, what does pytest report?               │
│     What does it report WITH strict=True?                       │
│     Which is safer and why?                                     │
│                                                                 │
│  4. A test fails. The assertion message alone doesn't tell you  │
│     enough. You want to see all local variables printed in the  │
│     failure output WITHOUT opening an interactive debugger.     │
│     What is the flag?                                           │
│                                                                 │
│  5. You placed a breakpoint() inside a test but the pdb prompt  │
│     never appears — pytest just hangs or skips it. What are     │
│     you missing?                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solutions:**

```python
# ── Answer 1 ──────────────────────────────────────────────────────
import sys

@pytest.mark.skipif(sys.platform != "linux", reason="Linux-only feature")
def test_linux_only_export():
    ...
# Correct decorator: skipif — the code CANNOT execute on non-Linux.
# xfail is wrong here: xfail RUNS the code and expects a failure.
# But if the code imports a Linux-only C extension, it will raise
# ImportError during collection, not a test assertion failure.
# skip prevents execution entirely.


# ── Answer 2 ──────────────────────────────────────────────────────
@pytest.mark.parametrize(
    "discount, expected",
    [
        (0,   100.0),
        (10,   90.0),
        (20,   80.0),
        pytest.param(          # ← Only this case is xfail
            -5, 100.0,
            marks=pytest.mark.xfail(reason="Bug #42: negative discount not rejected")
        ),
    ],
)
def test_discount_cases(discount, expected):
    ...
# pytest.param() attaches marks to a single data row.
# The other 3 rows are unaffected.


# ── Answer 3 ──────────────────────────────────────────────────────
# Without strict=True → XPASS (unexpected pass): pytest reports it
#   but the test suite EXIT CODE is still 0 (passes CI).
#   Dangerous: a fixed bug is silently treated as passing.
#
# With strict=True → XPASS → FAILED: exit code is non-zero.
#   CI breaks, forcing you to remove the xfail decorator.
#   This is safer: it makes sure fixed bugs are properly acknowledged.


# ── Answer 4 ──────────────────────────────────────────────────────
$ pytest -l           # or: pytest --showlocals
# Shows all local variable values at the failure frame.
# No interactive prompt. Read-only output. Fast to use.


# ── Answer 5 ──────────────────────────────────────────────────────
# You're missing the -s flag.
# pytest captures stdin/stdout by default.
# pdb requires interactive stdin — without -s, it can't receive input.
$ pytest -s tests/test_orders.py::test_name
```

---

# PART 3: FIXTURES

## 3.1 The Problem: Repeated Setup

**Look at what happens when tests share setup code:**

```python
# ❌ REPETITION: Same setup in every test
def test_order_total_no_discount():
    order = Order(item="Widget", quantity=10, unit_price=10.0)  # repeated
    assert calculate_total(order) == 100.0

def test_order_total_with_discount():
    order = Order(item="Widget", quantity=10, unit_price=10.0)  # repeated
    assert calculate_total(order, 20) == 80.0

def test_bulk_discount_applied():
    order = Order(item="Widget", quantity=10, unit_price=10.0)  # repeated
    assert apply_bulk_discount(order) == 85.0

def test_order_str_representation():
    order = Order(item="Widget", quantity=10, unit_price=10.0)  # repeated
    assert order.item == "Widget"
```

> "Four tests. The exact same `Order(...)` constructor in every single one. Now imagine 40 tests. And imagine the `Order` constructor changes — you add a required `currency` field. You'd have to update 40 lines."

```
┌─────────────────────────────────────────────────────────────────┐
│                THE SETUP REPETITION PROBLEM                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Factory analogy:                                               │
│                                                                 │
│  Imagine every QC inspector has to BUILD THEIR OWN test bench   │
│  from scratch before every single inspection.                   │
│                                                                 │
│  Inspector 1: Build bench → test engine → disassemble bench     │
│  Inspector 2: Build bench → test transmission → disassemble     │
│  Inspector 3: Build bench → test brakes → disassemble           │
│                                                                 │
│  That's insane. In a real factory, you have a SHARED TEST       │
│  BENCH that's set up once and every inspector uses it.          │
│                                                                 │
│  That's what fixtures are.                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 @pytest.fixture — The Test Workbench

**A fixture is a function that provides test data or setup.**

```python
import pytest
from src.orders import Order, calculate_total, apply_bulk_discount

@pytest.fixture
def standard_order() -> Order:
    """A standard 10-item order for testing."""
    return Order(item="Widget", quantity=10, unit_price=10.0)

# Tests REQUEST fixtures by putting them as function parameters:

def test_order_total_no_discount(standard_order: Order):
    assert calculate_total(standard_order) == 100.0

def test_order_total_with_discount(standard_order: Order):
    assert calculate_total(standard_order, 20) == 80.0

def test_bulk_discount_applied(standard_order: Order):
    assert apply_bulk_discount(standard_order) == 85.0

def test_order_item_name(standard_order: Order):
    assert standard_order.item == "Widget"
```

**How does this work?**

```
┌─────────────────────────────────────────────────────────────────┐
│                 HOW FIXTURES ARE INJECTED                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. pytest sees: def test_something(standard_order):            │
│                                                                 │
│  2. pytest checks: "Is there a fixture named standard_order?"   │
│                                                                 │
│  3. pytest calls the fixture function                           │
│                                                                 │
│  4. pytest passes the RETURN VALUE to your test                 │
│                                                                 │
│                                                                 │
│  @pytest.fixture                                                │
│  def standard_order():           ← pytest calls this            │
│      return Order(...)           ← gets this value              │
│                  │                                              │
│                  ▼                                              │
│  def test_something(standard_order):  ← value injected here     │
│      assert standard_order.quantity == 10                       │
│                                                                 │
│                                                                 │
│  This is DEPENDENCY INJECTION — the same pattern you'll         │
│  see in FastAPI's Depends() in Week 3!                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Multiple fixtures:**

```python
@pytest.fixture
def small_order() -> Order:
    return Order(item="Bolt", quantity=2, unit_price=1.50)

@pytest.fixture
def bulk_order() -> Order:
    return Order(item="Beam", quantity=100, unit_price=25.0)

@pytest.fixture
def free_order() -> Order:
    return Order(item="Sample", quantity=1, unit_price=0.0)

# A test can request MULTIPLE fixtures:
def test_bulk_is_more_expensive_than_small(
    small_order: Order, bulk_order: Order
):
    small_total = calculate_total(small_order)
    bulk_total = calculate_total(bulk_order)
    assert bulk_total > small_total
```

**Fixtures that depend on other fixtures:**

```python
@pytest.fixture
def standard_order() -> Order:
    return Order(item="Widget", quantity=10, unit_price=10.0)

@pytest.fixture
def discounted_total(standard_order: Order) -> float:
    """A fixture can use other fixtures too!"""
    return calculate_total(standard_order, discount_percent=15.0)

def test_discounted_total_value(discounted_total: float):
    assert discounted_total == 85.0
```

---

## 3.3 Fixture Scope

**By default, a fixture runs ONCE PER TEST. But you can control this.**

```python
@pytest.fixture(scope="function")   # ← DEFAULT: fresh for every test
def db_connection():
    print("Opening connection")
    conn = create_connection()
    return conn

@pytest.fixture(scope="module")     # ← Once per test FILE
def db_connection():
    print("Opening connection (once for this file)")
    conn = create_connection()
    return conn

@pytest.fixture(scope="session")    # ← Once for ALL tests in entire run
def db_connection():
    print("Opening connection (once for everything)")
    conn = create_connection()
    return conn
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    FIXTURE SCOPE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCOPE         CREATED          DESTROYED       USE WHEN        │
│  ──────────    ────────         ──────────      ────────        │
│  "function"    Before each      After each      Default.        │
│                test function    test function    Clean state.   │
│                                                                 │
│  "class"       Before each      After each      Grouping        │
│                test class       test class       related tests. │
│                                                                 │
│  "module"      Before first     After last       Expensive      │
│                test in file     test in file     per-file setup.│
│                                                                 │
│  "session"     Before first     After last       Very expensive │
│                test overall     test overall     global setup.  │
│                                                                 │
│                                                                 │
│  Factory analogy:                                               │
│  "function"  = Reset the test bench between EVERY part          │
│  "module"    = Set up bench once per SHIFT (batch of tests)     │
│  "session"   = Set up bench once per DAY (whole test run)       │
│                                                                 │
│  ⚠️ WARNING: broader scope = tests may SHARE state.             │
│  If test A modifies the fixture, test B sees the change.        │
│  Use "function" unless you have a good reason not to.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Yield Fixtures — Setup AND Teardown

**Connection to Context Managers (Week 1, Lecture 2):**

> "Remember context managers? `with open('f') as f:` — opens the file, then *guarantees* it closes when done? Yield fixtures are the exact same pattern for tests. Setup BEFORE yield, cleanup AFTER yield."

```python
# Context manager pattern you already know (Week 1):
from contextlib import contextmanager

@contextmanager
def managed_resource():
    print("Setup")         # ← Runs first
    resource = acquire()
    yield resource         # ← Test/code uses this value
    print("Teardown")      # ← Runs after, even if exception
    resource.release()

# Yield fixture — THE SAME PATTERN:
@pytest.fixture
def temp_file():
    # SETUP — runs before the test
    path = Path("test_output.tmp")
    path.write_text("test data")
    
    yield path              # ← Test receives this value
    
    # TEARDOWN — runs after the test, even if test fails!
    path.unlink()           # Clean up the temp file
```

**Visualize the lifecycle:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  YIELD FIXTURE LIFECYCLE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @pytest.fixture                                                │
│  def database():                                                │
│      db = connect()        ← SETUP: create the resource         │
│      db.begin_transaction()                                     │
│      │                                                          │
│      yield db              ← HAND TO TEST: test uses db         │
│      │                                                          │
│      db.rollback()         ← TEARDOWN: clean up (ALWAYS runs)   │
│      db.disconnect()                                            │
│                                                                 │
│                                                                 │
│  Timeline:                                                      │
│  ─────────                                                      │
│                                                                 │
│  [  SETUP  ] → [  yield  ] → [    TEST RUNS    ] → [ TEARDOWN ] │
│  connect()     return db      test_something(db)    rollback()  │
│  begin_txn()                  assert ...            disconnect()│
│                                                                 │
│  If test FAILS or RAISES → teardown STILL runs.                 │
│  Your resources are ALWAYS cleaned up. Guaranteed.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical example — temporary directory:**

```python
from pathlib import Path
import shutil

@pytest.fixture
def work_dir() -> Path:
    """Create a temporary working directory, clean up after."""
    path = Path("test_workspace")
    path.mkdir(exist_ok=True)       # SETUP
    
    yield path                       # Test uses this path
    
    shutil.rmtree(path)             # TEARDOWN: delete everything


def test_file_creation(work_dir: Path):
    output = work_dir / "results.json"
    output.write_text('{"status": "ok"}')
    assert output.exists()
    # After test: work_dir is deleted automatically!
```

---

## 3.5 conftest.py — The Shared Toolroom

**Fixtures in conftest.py are automatically available to ALL tests in the same directory and below.**

```
project/
├── tests/
│   ├── conftest.py          ← Fixtures here available EVERYWHERE
│   ├── test_orders.py       ← Can use fixtures from conftest.py
│   ├── test_discounts.py    ← Can use fixtures from conftest.py
│   └── api/
│       ├── conftest.py      ← Fixtures here available in api/ only
│       └── test_endpoints.py← Can use BOTH conftest.py files
```

```python
# tests/conftest.py — Shared fixtures for all tests

import pytest
from src.orders import Order

@pytest.fixture
def standard_order() -> Order:
    """Available to every test file without importing."""
    return Order(item="Widget", quantity=10, unit_price=10.0)

@pytest.fixture
def bulk_order() -> Order:
    """Available to every test file without importing."""
    return Order(item="Beam", quantity=100, unit_price=25.0)

@pytest.fixture
def empty_order() -> Order:
    return Order(item="Nothing", quantity=1, unit_price=0.0)
```

```python
# tests/test_orders.py — No import needed for fixtures!

def test_standard_order_total(standard_order: Order):  # ← Just use the name
    assert calculate_total(standard_order) == 100.0

def test_bulk_order_qualifies_for_discount(bulk_order: Order):  # ← From conftest
    assert bulk_order.quantity >= 10
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   conftest.py RULES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Name must be EXACTLY conftest.py (not conftest_orders.py)   │
│                                                                 │
│  2. NO IMPORTS needed — pytest finds and loads it automatically │
│                                                                 │
│  3. Scope is the DIRECTORY it's in (and all subdirectories)     │
│                                                                 │
│  4. You can have MULTIPLE conftest.py at different levels       │
│     └─ Fixtures in deeper conftest override shallower ones      │
│                                                                 │
│  5. Put SHARED fixtures here. Keep TEST-SPECIFIC fixtures       │
│     in the test file itself.                                    │
│                                                                 │
│  Factory analogy:                                               │
│  conftest.py = The shared toolroom.                             │
│  Every inspector grabs tools from here.                         │
│  Test-specific fixtures = the inspector's personal kit.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---
## 3.6 Built-in Fixtures — pytest's Standard Library

> "You've been building fixtures from scratch. pytest ships with powerful built-in fixtures for the most common test needs. You don't import them. Don't define them. Just declare them as test parameters — pytest injects them automatically."

### `tmp_path` — Isolated Temporary Directories

**Use when:** your code writes files, reads config files, or does any file-system I/O.

```python
# src/exporter.py — code that writes files
import json
from pathlib import Path
from src.orders import Order, apply_bulk_discount

def export_orders_to_json(orders: list[Order], output_dir: Path) -> Path:
    """Exports processed orders to a JSON file. Returns the file path."""
    results = []
    for order in orders:
        total = apply_bulk_discount(order)
        results.append({
            "item": order.item,
            "quantity": order.quantity,
            "total": round(total, 2),
        })

    out_path = output_dir / "orders_export.json"
    out_path.write_text(json.dumps(results, indent=2))
    return out_path
```

```python
# tests/test_exporter.py
import json
from pathlib import Path
from src.orders import Order
from src.exporter import export_orders_to_json

def test_export_creates_json_file(tmp_path: Path):
    # tmp_path is a fresh, empty directory unique to THIS test.
    # Automatically deleted when the test ends. No cleanup needed.
    orders = [
        Order(item="Widget", quantity=10, unit_price=10.0),
        Order(item="Gadget", quantity=5,  unit_price=20.0),
    ]

    result_path = export_orders_to_json(orders, output_dir=tmp_path)

    assert result_path.exists()
    data = json.loads(result_path.read_text())
    assert len(data) == 2
    assert data[0]["item"] == "Widget"
    assert data[0]["total"] == 85.0   # 15% bulk discount applied


def test_export_second_test_gets_clean_directory(tmp_path: Path):
    # This test gets a COMPLETELY DIFFERENT directory from the one above.
    # It does not see the file written by the first test.
    assert list(tmp_path.iterdir()) == []  # Empty — guaranteed
```

```
┌─────────────────────────────────────────────────────────────────┐
│               tmp_path vs tmp_path_factory                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  tmp_path          — function scope only. One dir per test.     │
│                      Use for: most cases.                       │
│                                                                 │
│  tmp_path_factory  — works with module/session scope fixtures.  │
│                      Use for: creating shared dirs in fixtures  │
│                      that outlive a single test.                │
│                                                                 │
│  @pytest.fixture(scope="module")                                │
│  def shared_dir(tmp_path_factory: pytest.TempPathFactory):      │
│      base = tmp_path_factory.mktemp("shared")                   │
│      (base / "seed.json").write_text('{"ready": true}')         │
│      return base                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### `monkeypatch` — Safely Patching the Test Environment

**The problem with directly modifying `os.environ` in tests:**

```python
# ❌ DANGEROUS: pollutes the global environment
def test_api_key_reading():
    os.environ["WEATHER_API_KEY"] = "test-key"  # Set globally!
    result = get_api_key()
    assert result == "test-key"
    # If test crashes before cleanup → key stays set for ALL remaining tests.
    # Race conditions if tests run in parallel. Side effects everywhere.
```

**The solution — `monkeypatch` reverts everything automatically:**

```python
# src/config.py
import os

def get_api_key() -> str:
    key = os.environ.get("WEATHER_API_KEY")
    if not key:
        raise ValueError("WEATHER_API_KEY environment variable is not set")
    return key

def get_base_url() -> str:
    return os.environ.get("API_BASE_URL", "https://api.weather.com")
```

```python
# tests/test_config.py
import os
import pytest
from src.config import get_api_key, get_base_url

def test_reads_api_key_from_environment(monkeypatch: pytest.MonkeyPatch):
    monkeypatch.setenv("WEATHER_API_KEY", "test-key-abc")
    # Change is ONLY visible for this test.
    assert get_api_key() == "test-key-abc"
    # After test: WEATHER_API_KEY is removed. No cleanup code needed.


def test_missing_key_raises_error(monkeypatch: pytest.MonkeyPatch):
    # Ensure the key is absent even on developer machines that have it set.
    monkeypatch.delenv("WEATHER_API_KEY", raising=False)
    # raising=False: don't raise an error if the key wasn't set to begin with.
    with pytest.raises(ValueError, match="WEATHER_API_KEY"):
        get_api_key()


def test_uses_default_base_url_when_not_set(monkeypatch: pytest.MonkeyPatch):
    monkeypatch.delenv("API_BASE_URL", raising=False)
    assert get_base_url() == "https://api.weather.com"


def test_uses_custom_base_url_when_set(monkeypatch: pytest.MonkeyPatch):
    monkeypatch.setenv("API_BASE_URL", "http://localhost:9000")
    assert get_base_url() == "http://localhost:9000"
```

**`monkeypatch.setattr` — replace any attribute on any module:**

```python
# Replace a function on a module for the duration of one test:
import src.orders as orders_module

def test_uses_patched_price_logic(monkeypatch: pytest.MonkeyPatch):
    def always_zero(order, discount_percent=0.0) -> float:
        return 0.0

    monkeypatch.setattr(orders_module, "calculate_total", always_zero)

    from src.orders import calculate_total, Order
    order = Order(item="Widget", quantity=10, unit_price=10.0)
    assert calculate_total(order) == 0.0
    # After test: original calculate_total is restored automatically.
```

```
┌─────────────────────────────────────────────────────────────────┐
│          monkeypatch vs unittest.mock.patch — WHEN EACH         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  monkeypatch                                                    │
│  ├─ A pytest FIXTURE (declare as parameter)                     │
│  ├─ Best for: env vars, simple attribute swaps, dict mutations  │
│  ├─ No verification methods (can't assert "was it called?")     │
│  └─ Auto-reverts all changes — zero cleanup code                │
│                                                                 │
│  unittest.mock.patch                                            │
│  ├─ A stdlib context manager or decorator                       │
│  ├─ Best for: replacing callables with MagicMock/AsyncMock      │
│  ├─ assert_called_once_with(), call_count, side_effect, etc.    │
│  └─ Use when you need to VERIFY how something was called        │
│                                                                 │
│  RULE:                                                          │
│  Environment/config changes → monkeypatch                       │
│  Replacing callables with verification → patch + MagicMock      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### `capsys` and `capfd` — Capturing stdout/stderr

**Use when:** your function calls `print()` or writes to `sys.stderr`, and you need to assert what was printed.

```python
# src/reporter.py
from src.orders import Order, calculate_total

def print_receipt(order: Order, discount_percent: float = 0.0) -> None:
    total = calculate_total(order, discount_percent)
    print("=== RECEIPT ===")
    print(f"Item:     {order.item}")
    print(f"Quantity: {order.quantity}")
    print(f"Total:    ${total:.2f}")
    if discount_percent:
        print(f"Discount: {discount_percent}% applied")
```

```python
# tests/test_reporter.py
import pytest
from src.reporter import print_receipt
from src.orders import Order

def test_receipt_shows_item_and_total(capsys: pytest.CaptureFixture[str]):
    order = Order(item="Widget", quantity=5, unit_price=10.0)
    print_receipt(order)

    captured = capsys.readouterr()   # Returns namedtuple: (out, err)

    assert "Widget" in captured.out
    assert "$50.00" in captured.out
    assert captured.err == ""        # Nothing written to stderr


def test_receipt_shows_discount_line(capsys: pytest.CaptureFixture[str]):
    order = Order(item="Widget", quantity=10, unit_price=10.0)
    print_receipt(order, discount_percent=15.0)

    captured = capsys.readouterr()
    assert "15.0% applied" in captured.out
    assert "$85.00" in captured.out


def test_no_discount_line_when_full_price(capsys: pytest.CaptureFixture[str]):
    order = Order(item="Bolt", quantity=2, unit_price=5.0)
    print_receipt(order)

    captured = capsys.readouterr()
    assert "Discount" not in captured.out
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    capsys vs capfd                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  capsys  — captures Python-level sys.stdout / sys.stderr        │
│            Covers: print(), sys.stdout.write(), logging         │
│            Use for: 99% of cases                                │
│                                                                 │
│  capfd   — captures OS file descriptor 1 and 2 directly         │
│            Covers: C extension output, subprocess writes        │
│            Use for: code that bypasses Python's stream objects  │
│                                                                 │
│  capsys is almost always what you want.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### `caplog` — Capturing and Asserting on Log Messages

**Use when:** your code emits log messages and you want to verify they are emitted at the correct level with the correct content.

```python
# src/orders.py — updated to include logging
import logging

logger = logging.getLogger(__name__)

def apply_bulk_discount(order: Order) -> float:
    """Orders of 10+ items get 15% off."""
    if order.quantity >= 10:
        logger.info("Bulk discount applied", extra={"qty": order.quantity})
        return calculate_total(order, discount_percent=15.0)

    logger.debug("No bulk discount: quantity below threshold",
                 extra={"qty": order.quantity})
    return calculate_total(order)
```

```python
# tests/test_orders.py
import logging
import pytest
from src.orders import Order, apply_bulk_discount

def test_bulk_order_logs_info_message(caplog: pytest.LogCaptureFixture):
    order = Order(item="Widget", quantity=15, unit_price=10.0)

    with caplog.at_level(logging.INFO):
        apply_bulk_discount(order)

    assert "Bulk discount applied" in caplog.text
    assert caplog.records[0].levelname == "INFO"


def test_small_order_logs_debug(caplog: pytest.LogCaptureFixture):
    order = Order(item="Bolt", quantity=3, unit_price=2.0)

    with caplog.at_level(logging.DEBUG):
        apply_bulk_discount(order)

    assert "below threshold" in caplog.text


def test_no_unexpected_errors_on_valid_order(caplog: pytest.LogCaptureFixture):
    """Verify the happy path emits no warnings or errors."""
    order = Order(item="Widget", quantity=10, unit_price=10.0)

    with caplog.at_level(logging.WARNING):
        apply_bulk_discount(order)

    error_records = [r for r in caplog.records if r.levelno >= logging.WARNING]
    assert len(error_records) == 0, f"Unexpected log entries: {error_records}"
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   caplog QUICK REFERENCE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  caplog.at_level(logging.DEBUG)  — set minimum capture level   │
│                                    (context manager OR call)    │
│  caplog.text         — all captured messages as one string      │
│  caplog.records      — list of logging.LogRecord objects        │
│  caplog.messages     — list of message strings only             │
│                                                                 │
│  Each LogRecord has:                                            │
│  .levelname  — "DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"  │
│  .levelno    — 10, 20, 30, 40, 50                               │
│  .message    — the formatted log message string                 │
│  .name       — logger name (e.g., "src.orders")                 │
│                                                                 │
│  ⚠️  Default capture level is WARNING.                          │
│  Always use caplog.at_level() or you won't capture DEBUG/INFO.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│          ✏️  PRACTICE CHECKPOINT — PART 3 (3.6)                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Your function writes a CSV file to the current working      │
│     directory. If two tests run this function, they would       │
│     overwrite each other's file. Which built-in fixture gives   │
│     each test its own isolated, auto-cleaned directory?         │
│                                                                 │
│  2. You call monkeypatch.setenv("API_KEY", "test") in test A.   │
│     Does test B see this value? What happens after test A ends? │
│                                                                 │
│  3. Write the test stub (fixture name, call, assertion) to      │
│     verify that a function named process() prints the word      │
│     "Done" to stdout.                                           │
│                                                                 │
│  4. Your caplog.records is empty even though the function you   │
│     are testing definitely calls logger.info(). What is the     │
│     single most likely cause?                                   │
│                                                                 │
│  5. When would you choose monkeypatch over unittest.mock.patch  │
│     to swap out a function? When would you choose patch()?      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solutions:**

```python
# ── Answer 1 ──────────────────────────────────────────────────────
# Use tmp_path. Each test gets a unique, empty directory.
# No teardown code needed — pytest deletes it after the test.
def test_csv_export(tmp_path: Path):
    result = export_to_csv(data, output_dir=tmp_path)
    assert result.exists()


# ── Answer 2 ──────────────────────────────────────────────────────
# Test B does NOT see the value. monkeypatch is scoped to function
# by default. After test A ends, the env var is REMOVED from
# os.environ. monkeypatch tracks every change and reverts them all.


# ── Answer 3 ──────────────────────────────────────────────────────
def test_process_prints_done(capsys: pytest.CaptureFixture[str]):
    process()
    captured = capsys.readouterr()
    assert "Done" in captured.out


# ── Answer 4 ──────────────────────────────────────────────────────
# You forgot caplog.at_level(). caplog only captures WARNING and
# above by default. logger.info() at level 20 is below this threshold.
#
# Fix:
def test_logs_info(caplog):
    with caplog.at_level(logging.INFO):   # ← This is required
        my_function()
    assert "expected message" in caplog.text


# ── Answer 5 ──────────────────────────────────────────────────────
# Use monkeypatch when:
#   - Swapping env vars (setenv/delenv)
#   - Replacing a simple attribute where you DON'T need to assert
#     how/whether it was called
#
# Use patch() (from unittest.mock) when:
#   - You need a MagicMock/AsyncMock to record calls
#   - You need assert_called_once_with(), call_count, side_effect
#   - You're replacing an async function (needs AsyncMock)
```
---

# PART 4: THE TESTING PYRAMID

## 4.1 Three Levels of Inspection

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE TESTING PYRAMID                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                        /\                                       │
│                       /  \                                      │
│                      / E2E\        Few, slow, expensive         │
│                     / Tests\       "Test-drive the car"         │
│                    /────────\                                   │
│                   /          \                                  │
│                  /Integration \   Some, moderate speed          │
│                 /    Tests     \  "Test the assembly"           │
│                /────────────────\                               │
│               /                  \                              │
│              /    Unit Tests      \  Many, fast, cheap          │
│             /                      \ "Test each part"           │
│            /________________________\                           │
│                                                                 │
│                                                                 │
│  MORE ◄──── Number of tests ────► FEWER                         │
│  FAST ◄──── Execution speed ────► SLOW                          │
│  CHEAP ◄─── Cost to maintain ───► EXPENSIVE                     │
│  NARROW ◄── Scope of what's tested ──► BROAD                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Unit Tests — Testing One Gear

**A unit test checks ONE function or method in ISOLATION.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      UNIT TESTS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What: Test a single function/method                            │
│  Scope: One "unit" of code                                      │
│  Speed: Milliseconds                                            │
│  Dependencies: None (all external things are mocked/faked)      │
│                                                                 │
│  Factory analogy: Testing a single gear.                        │
│  Does it have the right number of teeth?                        │
│  Is it the right diameter? Does it spin freely?                 │
│  You don't need the whole car to test a gear.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# Unit test: tests calculate_total in ISOLATION
# No database, no network, no file system, no other functions
def test_calculate_total_basic():
    order = Order(item="Widget", quantity=5, unit_price=10.0)
    assert calculate_total(order) == 50.0

def test_calculate_total_with_discount():
    order = Order(item="Widget", quantity=10, unit_price=10.0)
    assert calculate_total(order, discount_percent=20.0) == 80.0

def test_calculate_total_rejects_negative_quantity():
    order = Order(item="Widget", quantity=-1, unit_price=10.0)
    with pytest.raises(InvalidQuantityError):
        calculate_total(order)
```

**Characteristics of a good unit test:**

```
┌─────────────────────────────────────────────────────────────────┐
│                GOOD UNIT TESTS ARE:                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FAST       — Runs in milliseconds. No waiting for I/O.         │
│  ISOLATED   — Doesn't depend on database, network, files.       │
│  REPEATABLE — Same result every time. No randomness.            │
│  SELF-CHECK — Asserts pass/fail automatically. No manual look.  │
│  TIMELY     — Written close to the code it tests.               │
│                                                                 │
│  (Mnemonic: FIRST)                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

---
## 4.3 Integration Tests — Testing the Assembly (ADDITION)

> **ADD the following after the existing 4.3 content, before 4.4:**

**A complete, real working integration test — multiple modules + real file system:**

```python
# src/order_processor.py  ← New source file (define this first)
import json
import logging
from pathlib import Path
from src.orders import Order, apply_bulk_discount, calculate_total

logger = logging.getLogger(__name__)

def process_and_export(order: Order, output_dir: Path) -> Path:
    """
    Full pipeline: compute total, write receipt to file.
    Integration point: orders module + file system.
    """
    total = apply_bulk_discount(order)
    original = order.quantity * order.unit_price
    discount_pct = round((1 - total / original) * 100) if total < original else 0

    receipt = {
        "item": order.item,
        "quantity": order.quantity,
        "unit_price": order.unit_price,
        "total": round(total, 2),
        "discount_percent": discount_pct,
    }

    out_path = output_dir / f"receipt_{order.item.lower().replace(' ', '_')}.json"
    out_path.write_text(json.dumps(receipt, indent=2))
    logger.info("Receipt written", extra={"path": str(out_path)})
    return out_path
```

```python
# tests/test_order_processor.py — Real integration tests, zero mocks

import json
from pathlib import Path
import pytest
from src.orders import Order
from src.order_processor import process_and_export


# ─── Integration: orders module + file system ─────────────────────

def test_bulk_order_receipt_file_is_created(tmp_path: Path):
    """
    Integration test: process_and_export calls apply_bulk_discount,
    then writes to the REAL file system. No mocking of either.
    If either module has a bug, this test catches it.
    """
    order = Order(item="Server", quantity=15, unit_price=200.0)

    receipt_path = process_and_export(order, output_dir=tmp_path)

    # File was created at the expected path:
    assert receipt_path.exists()
    assert receipt_path.name == "receipt_server.json"

    # Content is correct (integration of both modules):
    data = json.loads(receipt_path.read_text())
    assert data["item"] == "Server"
    assert data["total"] == 2550.0      # 15% off 3000.00
    assert data["discount_percent"] == 15


def test_small_order_no_discount_in_file(tmp_path: Path):
    """Integration: small order path — no discount applied, file reflects this."""
    order = Order(item="Bolt", quantity=3, unit_price=2.50)

    receipt_path = process_and_export(order, output_dir=tmp_path)

    data = json.loads(receipt_path.read_text())
    assert data["total"] == 7.50
    assert data["discount_percent"] == 0


def test_two_different_orders_create_two_files(tmp_path: Path):
    """Integration: multiple calls each produce a separate receipt file."""
    widget_order = Order(item="Widget", quantity=10, unit_price=10.0)
    gadget_order = Order(item="Gadget", quantity=20, unit_price=5.0)

    path_1 = process_and_export(widget_order, output_dir=tmp_path)
    path_2 = process_and_export(gadget_order, output_dir=tmp_path)

    assert path_1 != path_2         # Different files
    assert path_1.exists()
    assert path_2.exists()
    assert len(list(tmp_path.iterdir())) == 2
```

```
┌─────────────────────────────────────────────────────────────────┐
│             WHY THIS IS AN INTEGRATION TEST                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  It connects TWO real components:                               │
│  1. src.orders module (discount logic)                          │
│  2. The operating system's file system                          │
│                                                                 │
│  Nothing is mocked. If either component is broken, this fails.  │
│  A unit test for process_and_export WOULD mock apply_bulk_       │
│  discount and file writes — but then you'd never know if the    │
│  INTEGRATION between them is correct.                           │
│                                                                 │
│  Unit tests → "Does this gear work alone?"                      │
│  Integration tests → "Does this gear mesh with that one?"       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 End-to-End Tests — Test-Driving the Car (UPDATED)

> **The following REPLACES the existing 4.4 section content from the `test_full_order_pipeline` example onward. The introductory box and factory analogy text are unchanged.**

**Define the components the E2E test will use:**

```python
# src/processor.py  ← New source file
from src.orders import Order, apply_bulk_discount

def generate_receipt(order: Order, total: float) -> dict[str, object]:
    """
    Generates a user-facing receipt dictionary.
    This is the output the user actually sees.
    """
    original = order.quantity * order.unit_price
    is_discounted = total < original
    discount_pct = round((1 - total / original) * 100) if is_discounted else 0

    return {
        "item": order.item,
        "quantity": order.quantity,
        "unit_price": order.unit_price,
        "discount_applied": f"{discount_pct}%" if discount_pct else "0%",
        "total": round(total, 2),
        "status": "confirmed",
    }
```

**E2E test — the complete user scenario, no mocks, all real:**

```python
# tests/test_e2e_order.py
import json
from pathlib import Path
import pytest
from src.orders import Order, apply_bulk_discount
from src.processor import generate_receipt
from src.order_processor import process_and_export


def test_e2e_bulk_laptop_order(tmp_path: Path):
    """
    E2E: A customer places a large laptop order.
    System applies bulk discount, generates receipt, exports to file.
    We verify the COMPLETE user-facing outcome — not individual components.

    If this fails: the bug is SOMEWHERE in the pipeline.
    (Use your unit tests to pinpoint exactly where.)
    """
    # ─── Step 1: Customer places order ────────────────────────────
    order = Order(item="Laptop", quantity=20, unit_price=999.99)

    # ─── Step 2: System applies discount policy ───────────────────
    total = apply_bulk_discount(order)

    # ─── Step 3: System generates user-facing receipt ─────────────
    receipt = generate_receipt(order, total)

    # ─── Step 4: System exports receipt to file ───────────────────
    receipt_path = process_and_export(order, output_dir=tmp_path)

    # ─── Step 5: Assert the complete user-facing outcome ──────────
    # Receipt content — what the customer sees:
    assert receipt["item"] == "Laptop"
    assert receipt["quantity"] == 20
    assert receipt["discount_applied"] == "15%"
    assert receipt["total"] == pytest.approx(16999.83)   # 20 × 999.99 × 0.85
    assert receipt["status"] == "confirmed"

    # File export — what operations team sees:
    assert receipt_path.exists()
    file_data = json.loads(receipt_path.read_text())
    assert file_data["total"] == pytest.approx(16999.83, abs=0.01)


def test_e2e_small_order_no_discount():
    """
    E2E: A customer places a small order (no discount threshold).
    End result: full price receipt, confirmed.
    """
    order = Order(item="USB Cable", quantity=2, unit_price=15.00)
    total = apply_bulk_discount(order)
    receipt = generate_receipt(order, total)

    # No discount — customer pays full price:
    assert receipt["total"] == 30.0
    assert receipt["discount_applied"] == "0%"
    assert receipt["status"] == "confirmed"
```

```
┌─────────────────────────────────────────────────────────────────┐
│               E2E vs INTEGRATION — WHAT'S THE DIFFERENCE?       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Integration test (4.3 example):                                │
│  → Tests that two specific components work together             │
│  → process_and_export: orders module + file system              │
│  → Fails pinpoint: "the file write OR the discount logic broke" │
│                                                                 │
│  E2E test (this section):                                       │
│  → Exercises the ENTIRE pipeline from user action to outcome    │
│  → order entry → discount → receipt → export                    │
│  → Fails broadly: "something in the end-to-end flow is broken"  │
│                                                                 │
│  Factory analogy:                                               │
│  Integration = test engine bolts onto transmission              │
│  E2E = get in the finished car and drive it 10 miles            │
│                                                                 │
│  ↑ Preview: In Week 3-4, this becomes an HTTP request into      │
│  FastAPI → database → response body assertion. Same structure.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 The Pyramid Shape — Why More Unit, Fewer E2E

```
┌─────────────────────────────────────────────────────────────────┐
│                WHY THE PYRAMID SHAPE?                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  UNIT TESTS (base — MANY):                                      │
│  ├─ Run in milliseconds                                         │
│  ├─ Easy to write, easy to debug when they fail                 │
│  ├─ Pinpoint EXACT function that broke                          │
│  └─ Hundreds of them cost almost nothing to run                 │
│                                                                 │
│  INTEGRATION TESTS (middle — SOME):                             │
│  ├─ Run in seconds                                              │
│  ├─ Catch "works alone, fails together" bugs                    │
│  ├─ Harder to debug: which part caused the failure?             │
│  └─ Need more setup, sometimes real databases                   │
│                                                                 │
│  E2E TESTS (top — FEW):                                         │
│  ├─ Run in seconds to minutes                                   │
│  ├─ Catch "the whole thing doesn't work" bugs                   │
│  ├─ Hardest to debug: failure could be ANYWHERE                 │
│  └─ Fragile: break when UI changes, slow to run                 │
│                                                                 │
│                                                                 │
│  PRACTICAL RATIO (rule of thumb):                               │
│  ├─ 70% unit tests                                              │
│  ├─ 20% integration tests                                       │
│  └─ 10% E2E tests                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visualize the debugging difference:**

```
When a UNIT TEST fails:
  "test_calculate_total_with_discount FAILED"
  → The bug is in calculate_total(). Go look at it. 🎯

When an INTEGRATION TEST fails:
  "test_bulk_discount_full_flow FAILED"
  → Bug is in apply_bulk_discount() OR calculate_total(). 
    Check both. 🔍🔍

When an E2E TEST fails:
  "test_full_order_pipeline FAILED"
  → Bug could be ANYWHERE: order creation, discount logic, 
    receipt generation, database write... 🔍🔍🔍🔍🔍
    Good luck.
```

> "This is why you want MANY unit tests. When something breaks, you want to know EXACTLY where. A failing unit test is a GPS coordinate. A failing E2E test is 'somewhere in this country.'"

---

# PART 5: TEST DOUBLES

## 5.1 The Isolation Problem

**The problem: your code depends on things you can't control.**

```python
# src/weather.py
import httpx

async def get_temperature(city: str) -> float:
    """Fetch current temperature from a weather API."""
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://api.weather.com/v1/current?city={city}"
        )
        data = response.json()
        return data["temperature"]
```

**Try to test this:**

```python
# tests/test_weather.py

# ❌ PROBLEM: This test hits the REAL weather API!
async def test_get_temperature():
    temp = await get_temperature("London")
    assert isinstance(temp, float)
    # What if the API is down? Test fails.
    # What if the temperature changes? Test is flaky.
    # What if you have no internet? Test fails.
    # What if the API has rate limits? Test blocked.
    # What if the API costs money? Test is expensive.
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE ISOLATION PROBLEM                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your code talks to EXTERNAL things:                            │
│  ├─ APIs (weather, payment, email)                              │
│  ├─ Databases (PostgreSQL, Redis)                               │
│  ├─ File system (reading/writing files)                         │
│  ├─ Time (datetime.now(), time.time())                          │
│  └─ Random values (random.randint())                            │
│                                                                 │
│  You CAN'T control these in tests:                              │
│  ├─ APIs go down, return different data, cost money             │
│  ├─ Databases need setup, are slow                              │
│  ├─ Files might not exist on the test machine                   │
│  ├─ Time is always different                                    │
│  └─ Random is... random                                         │
│                                                                 │
│  Solution: REPLACE the real thing with a FAKE thing in tests.   │
│                                                                 │
│  Factory analogy:                                               │
│  To test if the steering wheel MOUNT works, you don't need      │
│  the real engine running. Use a DUMMY engine block —            │
│  same shape, same bolt holes, but no moving parts.              │
│  You're testing the MOUNT, not the engine.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Mocks, Stubs, and Fakes — The Differences

```
┌─────────────────────────────────────────────────────────────────┐
│              TEST DOUBLES: THE FAMILY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STUB                                                           │
│  ────                                                           │
│  Returns a pre-programmed answer. Doesn't check how it's        │
│  called. Like a mannequin in a car seat for crash testing —     │
│  it's there to fill the space, nothing more.                    │
│                                                                 │
│  "When someone calls get_temperature(), just return 20.5."      │
│                                                                 │
│                                                                 │
│  MOCK                                                           │
│  ────                                                           │
│  Like a stub, but also RECORDS how it was called.               │
│  You can verify: "Was this function called? With what args?     │
│  How many times?" Like a crash test dummy with SENSORS —        │
│  fills the space AND measures the impact.                       │
│                                                                 │
│  "Return 20.5, AND I want to verify you called it with          │
│   city='London' exactly once."                                  │
│                                                                 │
│                                                                 │
│  FAKE                                                           │
│  ────                                                           │
│  A working implementation, but simplified.                      │
│  Like using a go-kart engine instead of a V8 for testing        │
│  the fuel line — it actually runs, just simpler.                │
│                                                                 │
│  "Instead of a real database, use an in-memory dict."           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In code:**

```python
# STUB — just returns a value
async def stub_get_temperature(city: str) -> float:
    return 20.5  # Always returns this. Doesn't care about city.


# FAKE — simplified but functional implementation
class FakeWeatherDB:
    """In-memory fake. Actually stores/retrieves data, but no real DB."""
    def __init__(self) -> None:
        self._data: dict[str, float] = {}
    
    async def save_temperature(self, city: str, temp: float) -> None:
        self._data[city] = temp
    
    async def get_temperature(self, city: str) -> float | None:
        return self._data.get(city)


# MOCK — Python's unittest.mock handles this (covered next section)
```

---

## 5.3 unittest.mock.patch — Swapping Real for Fake

**`patch` temporarily replaces a real object with a fake one during a test.**

```python
# src/weather.py
import httpx

async def get_temperature(city: str) -> float:
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://api.weather.com/v1/current?city={city}"
        )
        data = response.json()
        return data["temperature"]

async def get_weather_report(city: str) -> str:
    temp = await get_temperature(city)
    if temp > 30:
        return f"{city}: Hot ({temp}°C)"
    elif temp > 15:
        return f"{city}: Mild ({temp}°C)"
    else:
        return f"{city}: Cold ({temp}°C)"
```

**Testing `get_weather_report` without hitting the real API:**

```python
# tests/test_weather.py
from unittest.mock import patch, AsyncMock
import pytest
from src.weather import get_weather_report

@pytest.mark.asyncio
async def test_hot_weather_report():
    # patch REPLACES get_temperature with a fake during this test
    with patch("src.weather.get_temperature", new_callable=AsyncMock) as mock_temp:
        mock_temp.return_value = 35.0  # Stub: always return 35
        
        result = await get_weather_report("Dubai")
        
        assert result == "Dubai: Hot (35.0°C)"
        mock_temp.assert_called_once_with("Dubai")  # Mock: verify it was called

@pytest.mark.asyncio
async def test_cold_weather_report():
    with patch("src.weather.get_temperature", new_callable=AsyncMock) as mock_temp:
        mock_temp.return_value = 5.0
        
        result = await get_weather_report("Moscow")
        
        assert result == "Moscow: Cold (5.0°C)"
```

**Visualize what `patch` does:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    HOW PATCH WORKS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NORMAL EXECUTION:                                              │
│                                                                 │
│  get_weather_report("London")                                   │
│       │                                                         │
│       ├─→ get_temperature("London")                             │
│       │        │                                                │
│       │        └─→ httpx.get("https://api.weather.com/...")     │
│       │                │                                        │
│       │                └─→ 🌐 REAL INTERNET REQUEST             │
│       │                         │                               │
│       │                ◀── {"temperature": 18.5}                │
│       │        ◀── 18.5                                         │
│       ◀── "London: Mild (18.5°C)"                               │
│                                                                 │
│                                                                 │
│  WITH PATCH:                                                    │
│                                                                 │
│  get_weather_report("London")                                   │
│       │                                                         │
│       ├─→ get_temperature("London")  ← INTERCEPTED!             │
│       │        │                                                │
│       │        └─→ mock(return_value=20.0) ← FAKE, NO INTERNET  │
│       │        ◀── 20.0                                         │
│       ◀── "London: Mild (20.0°C)"                               │
│                                                                 │
│  The real get_temperature NEVER RUNS.                           │
│  No network. No API. Instant. Deterministic.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**CRITICAL: The path in `patch()` is WHERE THE THING IS USED, not where it's defined:**

```python
# ❌ WRONG: patching where the function is DEFINED
with patch("src.weather_api.get_temperature"):  # Where it lives
    ...

# ✅ CORRECT: patching where the function is IMPORTED/USED
with patch("src.weather.get_temperature"):  # Where it's used
    ...
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   PATCH TARGET RULE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  patch("module_where_its_LOOKED_UP.name")                       │
│                                                                 │
│  If weather.py does:                                            │
│      from weather_api import get_temperature                    │
│                                                                 │
│  Then weather.py has its OWN reference to get_temperature.      │
│  You must patch THAT reference:                                 │
│      patch("src.weather.get_temperature")  ✅                   │
│                                                                 │
│  NOT the original:                                              │
│      patch("src.weather_api.get_temperature")  ❌               │
│                                                                 │
│  This is the #1 source of "why isn't my mock working?"          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Decorator form of patch (cleaner for some cases):**

```python
# Instead of `with patch(...)`, you can use it as a decorator:

@patch("src.weather.get_temperature", new_callable=AsyncMock)
@pytest.mark.asyncio
async def test_mild_weather_report(mock_temp: AsyncMock):
    mock_temp.return_value = 20.0
    
    result = await get_weather_report("London")
    
    assert result == "London: Mild (20.0°C)"
    # Note: decorators inject mocks BOTTOM-UP
    # @patch at the bottom → first parameter after self/cls
```

---

## 5.4 MagicMock and AsyncMock

**`MagicMock` — the Swiss army knife of test doubles:**

```python
from unittest.mock import MagicMock, AsyncMock

# MagicMock accepts ANY attribute access or method call:
mock = MagicMock()
mock.anything          # Works (returns another MagicMock)
mock.whatever(1, 2, 3) # Works (returns another MagicMock)
mock.deep.nested.call()# Works

# You control what it returns:
mock.get_user.return_value = {"id": 1, "name": "Alice"}
result = mock.get_user(42)
print(result)  # {"id": 1, "name": "Alice"}

# You can verify how it was called:
mock.get_user.assert_called_once_with(42)
mock.get_user.assert_called_with(42)
assert mock.get_user.call_count == 1
```

**`AsyncMock` — for async functions (Connection to Week 1, Lecture 3):**

```python
# AsyncMock works like MagicMock, but returns awaitables:
async_mock = AsyncMock()
async_mock.return_value = {"temperature": 20.5}

# Can be awaited:
result = await async_mock("London")  # Works!
print(result)  # {"temperature": 20.5}

# Same verification methods:
async_mock.assert_called_once_with("London")
```

**When to use which:**

```
┌─────────────────────────────────────────────────────────────────┐
│              MagicMock vs AsyncMock                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Replacing a sync function:     Use MagicMock                   │
│  Replacing an async function:   Use AsyncMock                   │
│                                                                 │
│  With patch():                                                  │
│  ├─ patch("mod.sync_func")                  → auto MagicMock    │
│  └─ patch("mod.async_func",                                     │
│          new_callable=AsyncMock)             → AsyncMock        │
│                                                                 │
│  Rule: if the original function is async def,                   │
│  your mock MUST be AsyncMock, or await will fail.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical example — mocking an async dependency:**

```python
# src/aggregator.py
from src.weather import get_temperature

async def get_avg_temperature(cities: list[str]) -> float:
    """Fetch temperatures for multiple cities, return the average."""
    temps: list[float] = []
    for city in cities:
        temp = await get_temperature(city)
        temps.append(temp)
    return sum(temps) / len(temps)
```

```python
# tests/test_aggregator.py
from unittest.mock import patch, AsyncMock
import pytest
from src.aggregator import get_avg_temperature

@pytest.mark.asyncio
async def test_average_of_three_cities():
    with patch("src.aggregator.get_temperature", new_callable=AsyncMock) as mock:
        # Make mock return different values for each call
        mock.side_effect = [20.0, 30.0, 10.0]
        
        result = await get_avg_temperature(["London", "Dubai", "Oslo"])
        
        assert result == pytest.approx(20.0)
        assert mock.call_count == 3
        mock.assert_any_call("London")
        mock.assert_any_call("Dubai")
        mock.assert_any_call("Oslo")
```

**`side_effect` — controlling sequential return values or raising errors:**

```python
# Return different values on each call:
mock.side_effect = [20.0, 30.0, 10.0]
await mock("a")  # 20.0
await mock("b")  # 30.0
await mock("c")  # 10.0

# Raise an exception:
mock.side_effect = ConnectionError("API down")
await mock("London")  # raises ConnectionError!

# Use a function for dynamic behavior:
def fake_temp(city: str) -> float:
    return {"London": 15.0, "Dubai": 40.0}.get(city, 20.0)

mock.side_effect = fake_temp
await mock("Dubai")   # 40.0
await mock("London")  # 15.0
await mock("Paris")   # 20.0 (default)
```

---

## 5.5 When Mocking Goes Wrong

### Mistake 1: Mocking too much

```python
# ❌ BAD: You're not testing ANYTHING real anymore
async def test_over_mocked():
    with patch("src.orders.calculate_total") as mock_calc:
        mock_calc.return_value = 100.0  # Fake answer
        
        result = calculate_total(some_order)  # Calling the MOCK, not the real function!
        assert result == 100.0  # Of course it's 100.0, you told it to return that!
        # This test proves nothing.

# ✅ GOOD: Only mock the EXTERNAL dependency
async def test_report_logic():
    with patch("src.weather.get_temperature", new_callable=AsyncMock) as mock:
        mock.return_value = 35.0  # Fake the external API
        
        result = await get_weather_report("Dubai")  # Test the REAL logic
        assert result == "Dubai: Hot (35.0°C)"       # This tests string formatting
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    MOCKING RULE OF THUMB                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Mock at the BOUNDARY of your system.                           │
│                                                                 │
│  YOUR CODE         │  THE OUTSIDE WORLD                         │
│  ─────────         │  ─────────────────                         │
│  Logic ✅          │  Network calls → MOCK                      │
│  Calculations ✅   │  Database queries → MOCK                   │
│  Data transforms ✅│  File system → MOCK                        │
│  Formatting ✅     │  Time/randomness → MOCK                    │
│                    │                                            │
│  Test YOUR code for real. Mock THEIR code.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Mistake 2: Tests that mirror the implementation

```python
# ❌ BAD: This test will break if you refactor the internals
async def test_too_coupled_to_implementation():
    with patch("src.weather.httpx.AsyncClient.get") as mock_get:
        mock_response = MagicMock()
        mock_response.json.return_value = {"temperature": 20.0}
        mock_get.return_value = mock_response
        
        result = await get_temperature("London")
        
        # Testing IMPLEMENTATION DETAILS:
        mock_get.assert_called_with(
            "https://api.weather.com/v1/current?city=London"
        )
        # If you change the URL format, this test breaks
        # even if the function still works correctly!

# ✅ GOOD: Test the BEHAVIOR, not the implementation
async def test_returns_temperature_for_city():
    with patch("src.weather.get_temperature", new_callable=AsyncMock) as mock:
        mock.return_value = 20.0
        
        result = await get_weather_report("London")
        
        assert "20.0°C" in result  # Test what the USER sees
```

### Mistake 3: Forgetting to use AsyncMock for async functions

```python
# ❌ WRONG: Regular MagicMock can't be awaited!
with patch("src.weather.get_temperature") as mock:
    mock.return_value = 20.0
    result = await get_weather_report("London")
    # TypeError: object MagicMock can't be used in 'await' expression

# ✅ CORRECT: Use AsyncMock for async functions
with patch("src.weather.get_temperature", new_callable=AsyncMock) as mock:
    mock.return_value = 20.0
    result = await get_weather_report("London")  # Works!
```

---

# PART 6: TEST-DRIVEN DEVELOPMENT

## 6.1 Red-Green-Refactor — The TDD Cycle

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE TDD CYCLE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│              ┌──────────┐                                       │
│              │  🔴 RED  │                                       │
│              │  Write a │                                       │
│              │  failing │                                       │
│              │  test    │                                       │
│              └────┬─────┘                                       │
│                   │                                             │
│                   ▼                                             │
│  ┌──────────┐          ┌──────────────┐                         │
│  │🔵REFACTOR│◀─────────│ 🟢 GREEN     │                          │
│  │ Clean up │          │ Write the    │                         │
│  │ the code │          │ MINIMUM code │                         │
│  │ (tests   │          │ to pass the  │                         │
│  │  still   │          │ test         │                         │
│  │  pass)   │──────────▶              │                         │
│  └──────────┘          └──────────────┘                         │
│       │                                                         │
│       └───────── Repeat ──────────▶ 🔴 next test                │
│                                                                 │
│                                                                 │
│  🔴 RED:      Write a test. Run it. It MUST fail.               │
│               (If it passes, your test is wrong.)               │
│                                                                 │
│  🟢 GREEN:    Write the SIMPLEST code that makes the            │
│               test pass. No more. Ugly is fine.                 │
│                                                                 │
│  🔵 REFACTOR: Clean up. Improve names, remove duplication.      │
│               Run tests again — still green? Good.              │
│                                                                 │
│  Factory analogy:                                               │
│  Write the quality spec BEFORE manufacturing the part.          │
│  Build the part to PASS the spec. Then polish it.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 TDD Walkthrough — Live Coding

**Task: Build a function that formats prices with currency symbols.**

**Cycle 1 — 🔴 RED:**

```python
# tests/test_formatter.py
from src.formatter import format_price

def test_format_usd():
    assert format_price(19.99, "USD") == "$19.99"
```

```bash
$ pytest tests/test_formatter.py
E   ModuleNotFoundError: No module named 'src.formatter'
# 🔴 RED — good! The function doesn't even exist yet.
```

**Cycle 1 — 🟢 GREEN (minimum code to pass):**

```python
# src/formatter.py
def format_price(amount: float, currency: str) -> str:
    return "$19.99"  # Yes, this is cheating. That's the point.
```

```bash
$ pytest tests/test_formatter.py
PASSED  # 🟢 GREEN
```

> "Wait, that's clearly wrong!" — Exactly. TDD says: **write another test to force better code.**

**Cycle 2 — 🔴 RED:**

```python
# Add another test that the hard-coded answer can't satisfy:
def test_format_usd_different_amount():
    assert format_price(5.00, "USD") == "$5.00"
```

```bash
$ pytest
FAILED  # assert '$19.99' == '$5.00'
# 🔴 RED — hard-coded answer exposed!
```

**Cycle 2 — 🟢 GREEN:**

```python
# src/formatter.py
def format_price(amount: float, currency: str) -> str:
    return f"${amount:.2f}"  # Now actually uses the amount
```

```bash
$ pytest
2 passed  # 🟢 GREEN
```

**Cycle 3 — 🔴 RED (new currency):**

```python
def test_format_eur():
    assert format_price(19.99, "EUR") == "€19.99"
```

```bash
$ pytest
FAILED  # assert '$19.99' == '€19.99'
# 🔴 RED — doesn't handle EUR yet
```

**Cycle 3 — 🟢 GREEN:**

```python
# src/formatter.py
CURRENCY_SYMBOLS: dict[str, str] = {
    "USD": "$",
    "EUR": "€",
    "GBP": "£",
}

def format_price(amount: float, currency: str) -> str:
    symbol = CURRENCY_SYMBOLS.get(currency, currency)
    return f"{symbol}{amount:.2f}"
```

```bash
$ pytest
3 passed  # 🟢 GREEN
```

**Cycle 3 — 🔵 REFACTOR:**

```python
# src/formatter.py — cleaned up, type hints, docstring
from typing import Final

CURRENCY_SYMBOLS: Final[dict[str, str]] = {
    "USD": "$",
    "EUR": "€",
    "GBP": "£",
}

class UnsupportedCurrencyError(Exception):
    """Raised when currency code is not recognized."""
    def __init__(self, currency: str):
        self.currency = currency
        super().__init__(f"Unsupported currency: {currency}")

def format_price(amount: float, currency: str) -> str:
    """Format a price with the appropriate currency symbol.
    
    Args:
        amount: The price amount (must be non-negative).
        currency: ISO 4217 currency code (e.g., 'USD', 'EUR').
    
    Returns:
        Formatted price string like '$19.99'.
    
    Raises:
        UnsupportedCurrencyError: If currency code is not recognized.
        ValueError: If amount is negative.
    """
    if amount < 0:
        raise ValueError(f"Price cannot be negative: {amount}")
    if currency not in CURRENCY_SYMBOLS:
        raise UnsupportedCurrencyError(currency)
    symbol = CURRENCY_SYMBOLS[currency]
    return f"{symbol}{amount:.2f}"
```

```bash
$ pytest
3 passed  # 🟢 Still green after refactoring!
# NOW add tests for the new error handling:
```

```python
def test_negative_price_raises_value_error():
    with pytest.raises(ValueError, match="negative"):
        format_price(-5.0, "USD")

def test_unknown_currency_raises_error():
    with pytest.raises(UnsupportedCurrencyError):
        format_price(10.0, "XYZ")
```

---

## 6.3 When TDD Helps, When It Doesn't

```
┌─────────────────────────────────────────────────────────────────┐
│                 TDD: WHEN TO USE IT                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TDD SHINES WHEN:                                               │
│  ├─ Requirements are clear ("given X, return Y")                │
│  ├─ You're building utilities, parsers, formatters              │
│  ├─ You're fixing a bug (write test that reproduces bug first)  │
│  ├─ You're building an API (test the contract before the code)  │
│  └─ You need confidence that edge cases are handled             │
│                                                                 │
│  TDD IS AWKWARD WHEN:                                           │
│  ├─ You're exploring / prototyping (don't know what to build)   │
│  ├─ The requirements keep changing every hour                   │
│  ├─ You're doing UI/visual work (hard to assert "looks good")   │
│  └─ You're learning a new library (experiment first, test later)│
│                                                                 │
│  PRAGMATIC RULE:                                                │
│  You don't ALWAYS have to do TDD. But you ALWAYS have to        │
│  have tests. Write them before OR after — just write them.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 7: PYTEST CONFIGURATION AND COVERAGE

## 7.1 Configuring pytest in pyproject.toml

**Connection to Week 1, Lecture 4:**

> "You set up pyproject.toml with uv to manage dependencies and project metadata. pytest also reads from pyproject.toml — it's the single source of truth for your entire project's tooling config. No separate pytest.ini needed."

```toml
# pyproject.toml — add this section

[tool.pytest.ini_options]

# Where pytest looks for tests (relative to project root):
testpaths = ["tests"]

# Patterns pytest recognises as test files/functions:
python_files  = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]

# Default flags applied to every `pytest` run:
addopts = """
    --tb=short
    -v
    --strict-markers
"""
# --tb=short     → shorter tracebacks (less noise)
# -v             → verbose: show each test name
# --strict-markers → fail if an unregistered marker is used (prevents typos)

# Register your custom markers (required with --strict-markers):
markers = [
    "slow: marks tests as slow (deselect with '-m not slow')",
    "integration: marks tests requiring real file/DB/network access",
    "smoke: marks critical path tests that always run",
    "unit: marks pure logic tests with no external dependencies",
    "wip: work in progress — do not block CI",
]

# Filter out specific warnings (suppress known library noise):
filterwarnings = [
    "ignore::DeprecationWarning:httpx",
    "error::pytest.PytestUnknownMarkWarning",
    # "error" treats unregistered markers as ERRORS — keeps your marker list clean
]

# pytest-asyncio mode (required if you use async tests):
asyncio_mode = "auto"
# "auto" → every async def test_ is automatically treated as an asyncio test.
# You no longer need @pytest.mark.asyncio on every async test function.
```

**Running tests with your config active:**

```bash
# All your addopts flags are applied automatically:
$ pytest
# Equivalent to: pytest --tb=short -v --strict-markers tests/

# Override addopts for a specific run:
$ pytest --tb=long     # Overrides --tb=short from addopts
$ pytest -q            # Quiet mode, overrides -v from addopts

# Verify your config is loaded correctly:
$ pytest --co -q       # --co = collect only, shows what would run
```

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT --strict-markers PROTECTS YOU FROM            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Without --strict-markers:                                      │
│                                                                 │
│  @pytest.mark.slwo   ← TYPO. pytest silently ignores it.        │
│  def test_something():                                          │
│      ...                                                        │
│  $ pytest -m "not slow"  ← This test RUNS despite the typo!    │
│  # You think it's filtered. It's not.                           │
│                                                                 │
│                                                                 │
│  With --strict-markers:                                         │
│                                                                 │
│  @pytest.mark.slwo   ← TYPO.                                    │
│  def test_something():                                          │
│      ...                                                        │
│  $ pytest                                                       │
│  ERROR: 'slwo' is not a registered mark.                        │
│  ← Caught immediately. You fix the typo.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.2 Measuring Coverage with pytest-cov

**Install:**

```bash
$ uv add --dev pytest-cov
```

**The basic command:**

```bash
# Measure coverage for the src/ package, show missing lines:
$ pytest --cov=src --cov-report=term-missing

# Output:
# ---------- coverage: platform linux, python 3.12 ----------
# Name                    Stmts   Miss  Cover   Missing
# -------------------------------------------------------
# src/config.py               8      2    75%   14-15
# src/order_processor.py     18      0   100%
# src/orders.py              24      3    88%   31, 44, 47
# src/reporter.py            10      1    90%   22
# -------------------------------------------------------
# TOTAL                      60      6    90%
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 READING THE COVERAGE REPORT                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Stmts   — total executable lines in the file                   │
│  Miss    — lines never executed by any test                     │
│  Cover   — (Stmts - Miss) / Stmts × 100%                        │
│  Missing — the EXACT line numbers that are not covered          │
│                                                                 │
│  "src/orders.py 88%  Missing: 31, 44, 47"                       │
│  → Go to lines 31, 44, 47 in orders.py                          │
│  → These are code paths no test ever exercises                  │
│  → Write a test that exercises those paths                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Generating an HTML report (for visual inspection):**

```bash
$ pytest --cov=src --cov-report=html

# Creates: htmlcov/index.html
# Open it in your browser.
# Lines highlighted RED → not covered.
# Lines highlighted GREEN → covered.
# Click any file to see exactly which lines are tested.
```

**Branch coverage — beyond just line coverage:**

```python
# src/orders.py
def apply_bulk_discount(order: Order) -> float:
    if order.quantity >= 10:           # ← Line 8
        return calculate_total(order, discount_percent=15.0)   # ← Line 9
    return calculate_total(order)      # ← Line 10
```

```bash
# Line coverage only: if you test with quantity=15, line 8 is "covered"
# even though the ELSE branch (quantity < 10) was never reached.

# Branch coverage catches this:
$ pytest --cov=src --cov-branch --cov-report=term-missing

# Missing     Partial
# 10          8->10       ← Line 8 TRUE branch executed. FALSE branch not.
```

**Enforcing a minimum coverage threshold:**

```bash
# Fail the entire test run if coverage drops below 80%:
$ pytest --cov=src --cov-fail-under=80

# Exit code 1 if coverage < 80. CI pipeline fails. Prevents regressions.
```

**Add to pyproject.toml so it runs automatically:**

```toml
[tool.pytest.ini_options]
addopts = """
    --tb=short
    -v
    --strict-markers
    --cov=src
    --cov-report=term-missing
    --cov-fail-under=80
"""
```

```
┌─────────────────────────────────────────────────────────────────┐
│            LINE COVERAGE vs BRANCH COVERAGE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LINE COVERAGE:                                                 │
│  "Was this line executed at least once?"                        │
│  Misses: un-taken branches within an if statement               │
│                                                                 │
│  BRANCH COVERAGE:                                               │
│  "Was every decision point's TRUE and FALSE path exercised?"    │
│  Catches: the elif branch you never test, the else you forgot   │
│                                                                 │
│  WHAT COVERAGE DOESN'T CATCH:                                   │
│  ├─ Logic errors in covered code (calculate_total bug from 1.1  │
│  │   would be on a covered line — coverage wouldn't find it)    │
│  ├─ Missing tests for requirements (untested features)          │
│  └─ Test quality (an assert True covers the line but proves     │
│     nothing)                                                    │
│                                                                 │
│  Coverage tells you what you HAVEN'T tested.                    │
│  It does NOT tell you that what you HAVE tested is correct.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│        ✏️  PRACTICE CHECKPOINT — PART 7                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. You run pytest and get:                                     │
│     "PytestUnknownMarkWarning: Unknown pytest.mark.slow"        │
│     What are TWO things you must do to fix this?                │
│                                                                 │
│  2. After adding --cov-fail-under=80 to addopts, your CI        │
│     pipeline starts failing even though all tests pass.         │
│     What does this mean, and what should you do?                │
│                                                                 │
│  3. Coverage says src/orders.py is 88% covered.                 │
│     Missing: lines 31, 44. You open line 31 and see:            │
│       if discount_percent < 0:                                  │
│           raise InvalidDiscountError(discount_percent)          │
│     What kind of test do you write to cover this line?          │
│                                                                 │
│  4. A teammate says "We have 95% line coverage — our code is    │
│     well tested." Is this statement fully accurate? Why/why not?│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solutions:**

```python
# ── Answer 1 ──────────────────────────────────────────────────────
# Two things required:
# 1. Register the marker in pyproject.toml:
#    [tool.pytest.ini_options]
#    markers = ["slow: marks tests that take > 1 second"]
#
# 2. Add --strict-markers to addopts to turn the warning into an error:
#    addopts = "--strict-markers"
# Without --strict-markers, the warning is just a warning.
# With it, using an unregistered marker immediately fails the run.


# ── Answer 2 ──────────────────────────────────────────────────────
# It means your test suite has < 80% code coverage.
# Tests pass, but there are code paths never exercised by any test.
# What to do: run pytest --cov=src --cov-report=term-missing
# to see WHICH lines are uncovered, then write tests for them.
# Do NOT just lower the threshold — that defeats the purpose.


# ── Answer 3 ──────────────────────────────────────────────────────
# Write a test that passes a negative discount_percent:
def test_negative_discount_raises_error():
    order = Order(item="Widget", quantity=5, unit_price=10.0)
    with pytest.raises(InvalidDiscountError):
        calculate_total(order, discount_percent=-10.0)
# This exercises the if discount_percent < 0 branch (line 31).


# ── Answer 4 ──────────────────────────────────────────────────────
# Not fully accurate. "95% line coverage" means 95% of lines
# were EXECUTED — not that the logic on those lines is CORRECT.
#
# The calculate_total bug from Part 1 (returning discount amount
# instead of discounted price) would live on a COVERED line.
# Coverage would show 100% for that function.
# Coverage measures execution, not correctness.
# Assertions measure correctness. You need BOTH.
```

---

# PART 8: PRACTICAL TESTING PATTERNS

## 8.1 Testing CLI Applications

**This is directly relevant to the mini-project: your Async CLI Weather/News Aggregator is a command-line application. Here is how to test it.**

### Using Click's CliRunner

**If your CLI is built with Click (the most common Python CLI framework):**

```python
# src/main.py — a minimal Click CLI for the weather aggregator
import asyncio
import click

@click.group()
def cli() -> None:
    """Weather and News Aggregator CLI."""
    pass

@cli.command()
@click.argument("city")
@click.option("--units", default="celsius", help="Temperature units.")
def weather(city: str, units: str) -> None:
    """Fetch current weather for CITY."""
    # In real code: calls async get_temperature(city)
    # For demo: synchronous simulation
    click.echo(f"Fetching weather for {city}...")
    click.echo(f"Temperature: 20°C ({units})")

@cli.command()
@click.argument("query")
@click.option("--count", default=5, help="Number of articles.")
def news(query: str, count: int) -> None:
    """Fetch top news for QUERY."""
    if count < 1 or count > 50:
        raise click.BadParameter("count must be between 1 and 50", param_hint="--count")
    click.echo(f"Top {count} articles for: {query}")
```

```python
# tests/test_cli.py
from click.testing import CliRunner
from src.main import cli

@pytest.fixture
def runner() -> CliRunner:
    """Provides a CLI test runner. Isolated per test."""
    return CliRunner()


# ─── Test basic invocation ─────────────────────────────────────────

def test_weather_command_outputs_city_name(runner: CliRunner):
    result = runner.invoke(cli, ["weather", "London"])

    assert result.exit_code == 0           # 0 = success
    assert "London" in result.output
    assert "Temperature" in result.output


def test_weather_command_with_units(runner: CliRunner):
    result = runner.invoke(cli, ["weather", "Tokyo", "--units", "fahrenheit"])

    assert result.exit_code == 0
    assert "fahrenheit" in result.output


# ─── Test error handling ───────────────────────────────────────────

def test_news_with_invalid_count(runner: CliRunner):
    result = runner.invoke(cli, ["news", "Python", "--count", "200"])

    assert result.exit_code != 0          # Non-zero = error
    assert "count must be between" in result.output


# ─── Test help output ─────────────────────────────────────────────

def test_cli_help_is_available(runner: CliRunner):
    result = runner.invoke(cli, ["--help"])

    assert result.exit_code == 0
    assert "Usage:" in result.output


# ─── Test with mocked external dependency ─────────────────────────

from unittest.mock import patch, AsyncMock

def test_weather_uses_real_api_when_key_present(
    runner: CliRunner, monkeypatch: pytest.MonkeyPatch
):
    monkeypatch.setenv("WEATHER_API_KEY", "test-key")

    with patch("src.main.get_temperature", new_callable=AsyncMock) as mock_temp:
        mock_temp.return_value = 22.5
        result = runner.invoke(cli, ["weather", "Berlin"])

    assert result.exit_code == 0
    # CliRunner captured all output — even from async callbacks
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   CliRunner REFERENCE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  runner.invoke(cli, ["command", "arg", "--flag", "value"])      │
│                                                                 │
│  Returns a Result object:                                       │
│  .exit_code   — 0 = success, non-zero = error                   │
│  .output      — everything printed to stdout (string)           │
│  .exception   — the Python exception, if one was raised         │
│  .exc_info    — full exception info tuple                       │
│                                                                 │
│  runner = CliRunner(mix_stderr=False)                           │
│  → Separates stdout and stderr (default: mixed into .output)    │
│  → Access separately: result.output (stdout), ???               │
│     Actually with mix_stderr=False:                             │
│     result.output = stdout only                                 │
│                                                                 │
│  runner = CliRunner(env={"MY_VAR": "value"})                    │
│  → Sets environment variables just for this invoke              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**If your CLI uses Typer instead of Click:**

```python
# Typer has its own CliRunner:
from typer.testing import CliRunner
from src.main import app      # typer.Typer instance

runner = CliRunner()
result = runner.invoke(app, ["weather", "Paris"])
# API is identical to Click's CliRunner.
```

**If your CLI uses argparse (bare stdlib):**

```python
# For argparse, test the parsing logic directly — no CliRunner needed:
from src.main import build_parser

def test_argparse_city_argument():
    parser = build_parser()
    args = parser.parse_args(["weather", "--city", "Paris", "--units", "C"])
    assert args.city == "Paris"
    assert args.units == "C"

# Test the main function with mocked dependencies:
def test_main_with_mocked_api(monkeypatch):
    monkeypatch.setenv("WEATHER_API_KEY", "test")
    with patch("src.main.fetch_weather", new_callable=AsyncMock) as mock:
        mock.return_value = {"temp": 15.0}
        result = asyncio.run(run_weather_command(city="Paris"))
    assert result["temp"] == 15.0
```

---

## 8.2 Time-Dependent Testing

**The problem:** Code that calls `datetime.now()` returns different values every run. You can't write a stable assertion against "now".

```python
# src/orders.py — updated with timestamps
from datetime import datetime

@dataclass
class Order:
    item: str
    quantity: int
    unit_price: float
    created_at: datetime = None

    def __post_init__(self):
        if self.created_at is None:
            self.created_at = datetime.now()    # ← Non-deterministic

def is_order_recent(order: Order, within_seconds: int = 3600) -> bool:
    """Returns True if the order was created within the last hour."""
    age = (datetime.now() - order.created_at).total_seconds()
    return age <= within_seconds
```

```python
# ❌ BROKEN TEST: passes now, may fail in 1 hour
def test_order_is_recent():
    order = Order(item="Widget", quantity=5, unit_price=10.0)
    assert is_order_recent(order)   # True now, False in 61 minutes!
```

**The solution: `freezegun` — pin time to a fixed point:**

```bash
$ uv add --dev freezegun
```

```python
from freezegun import freeze_time
from datetime import datetime

# ─── As a decorator ──────────────────────────────────────────────

@freeze_time("2024-06-15 10:00:00")
def test_order_is_recent_at_creation():
    # datetime.now() returns 2024-06-15 10:00:00 inside this test.
    order = Order(item="Widget", quantity=5, unit_price=10.0)
    # order.created_at == datetime(2024, 6, 15, 10, 0, 0)
    assert is_order_recent(order)    # Age = 0 seconds. Always True.


@freeze_time("2024-06-15 10:00:00")
def test_old_order_is_not_recent():
    old_order = Order(
        item="Widget",
        quantity=5,
        unit_price=10.0,
        created_at=datetime(2024, 6, 15, 8, 0, 0)  # 2 hours ago
    )
    assert not is_order_recent(old_order)   # Age = 7200s > 3600s


# ─── As a context manager ─────────────────────────────────────────

def test_time_within_test():
    with freeze_time("2024-01-15 12:00:00"):
        order = Order(item="Widget", quantity=1, unit_price=5.0)
        assert order.created_at == datetime(2024, 1, 15, 12, 0, 0)

    # Outside the context manager: time is real again.
    order_after = Order(item="Bolt", quantity=1, unit_price=1.0)
    assert order_after.created_at != datetime(2024, 1, 15, 12, 0, 0)


# ─── Parametrize with frozen time ────────────────────────────────

@pytest.mark.parametrize("age_seconds, expected_recent", [
    (0,    True),      # Just created
    (3599, True),      # 1 second before cutoff
    (3600, True),      # Exactly at cutoff (within_seconds=3600, <= is True)
    (3601, False),     # 1 second past cutoff
    (86400, False),    # 1 day old
])
@freeze_time("2024-06-15 10:00:00")
def test_is_order_recent_boundary_cases(age_seconds: int, expected_recent: bool):
    from datetime import timedelta
    base = datetime(2024, 6, 15, 10, 0, 0)
    order = Order(
        item="Widget",
        quantity=1,
        unit_price=1.0,
        created_at=base - timedelta(seconds=age_seconds)
    )
    assert is_order_recent(order) == expected_recent
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   HOW freeze_time WORKS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  freeze_time patches datetime.datetime, datetime.date,          │
│  time.time, time.localtime, and time.gmtime in ALL modules       │
│  for the duration of the test/context.                          │
│                                                                 │
│  "2024-06-15 10:00:00" → ISO 8601 string                        │
│  datetime(2024, 6, 15, 10, 0, 0) → datetime object also works  │
│                                                                 │
│  ⚠️  freezegun patches at the Python level.                     │
│  OS-level time (subprocess calls, C extensions) is not frozen.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.3 Warnings Testing

**`pytest.warns` — the counterpart to `pytest.raises` for Python warnings:**

```python
import warnings
from src.orders import Order, calculate_total

# src/orders.py — add a deprecation warning to an old parameter:
def calculate_total(
    order: Order,
    discount_percent: float = 0.0,
    discount: float | None = None,   # ← Old parameter name, deprecated
) -> float:
    if discount is not None:
        warnings.warn(
            "The 'discount' parameter is deprecated. Use 'discount_percent'.",
            DeprecationWarning,
            stacklevel=2,
        )
        discount_percent = discount
    subtotal = order.quantity * order.unit_price
    return subtotal - subtotal * (discount_percent / 100)
```

```python
# tests/test_orders.py

def test_deprecated_discount_param_emits_warning():
    order = Order(item="Widget", quantity=5, unit_price=10.0)

    with pytest.warns(DeprecationWarning, match="'discount' parameter is deprecated"):
        result = calculate_total(order, discount=20.0)

    assert result == 40.0   # Verify it still works correctly


def test_modern_api_emits_no_deprecation_warning():
    order = Order(item="Widget", quantity=5, unit_price=10.0)

    # recwarn fixture: assert NO warnings are emitted
    with warnings.catch_warnings():
        warnings.simplefilter("error")   # Turn any warning into an error
        result = calculate_total(order, discount_percent=20.0)

    assert result == 40.0
```

```
┌─────────────────────────────────────────────────────────────────┐
│               pytest.warns REFERENCE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  with pytest.warns(DeprecationWarning):                         │
│      → Passes if DeprecationWarning is raised inside            │
│      → Fails if no warning is emitted                           │
│                                                                 │
│  with pytest.warns(DeprecationWarning, match="old parameter"):  │
│      → Also checks the warning message against a regex          │
│                                                                 │
│  Mirrors pytest.raises exactly. Same syntax, same patterns.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.4 Property-Based Testing — Brief Introduction

**The problem with hand-written test cases:** You can only think of the cases you already know about. Edge cases you don't imagine never get tested.

**Hypothesis** generates hundreds of test cases automatically and finds edge cases you'd never write yourself.

```bash
$ uv add --dev hypothesis
```

```python
from hypothesis import given, settings
from hypothesis import strategies as st

# Instead of: manually writing (0, 100, -1, 999, etc.)
# hypothesis generates them FOR you — and remembers cases that fail:

@given(
    quantity=st.integers(min_value=1, max_value=10_000),
    unit_price=st.floats(min_value=0.01, max_value=9_999.99),
    discount=st.floats(min_value=0.0, max_value=100.0),
)
def test_total_is_never_negative(quantity: int, unit_price: float, discount: float):
    """
    Property: the total of any valid order is never negative.
    hypothesis will try hundreds of combinations looking for a violation.
    """
    order = Order(item="Item", quantity=quantity, unit_price=unit_price)
    total = calculate_total(order, discount_percent=discount)
    assert total >= 0.0


@given(quantity=st.integers(min_value=10, max_value=1000))
def test_bulk_discount_total_always_less_than_full_price(quantity: int):
    """
    Property: bulk discount total is always less than full price.
    """
    order = Order(item="Item", quantity=quantity, unit_price=10.0)
    discounted = calculate_total(order, discount_percent=15.0)
    full_price  = calculate_total(order, discount_percent=0.0)
    assert discounted < full_price
```

```bash
$ pytest tests/test_hypothesis.py -v

tests/test_hypothesis.py::test_total_is_never_negative
  Trying 100 examples...   PASSED


# When hypothesis FINDS a failure:
# Falsifying example: quantity=1, unit_price=0.01, discount=100.0
# → It found the minimal reproducing case. You now have a new test case.
```

```
┌─────────────────────────────────────────────────────────────────┐
│            PROPERTY-BASED TESTING — WHEN TO USE IT              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE IT FOR:                                                    │
│  ├─ Mathematical properties (result always >= 0, monotonic)     │
│  ├─ Invariants (serialise → deserialise → same object)          │
│  └─ Boundary discovery (let hypothesis find the edge case)      │
│                                                                 │
│  NOT A REPLACEMENT for regular tests:                           │
│  ├─ Properties are hard to specify for business logic           │
│  └─ Slow compared to parametrize (runs 100+ examples)           │
│                                                                 │
│  Think of it as: parametrize, but the computer writes the       │
│  test data. Great complement to your existing test suite.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.5 Test Data Factories

**The problem:** As your test suite grows, every test manually constructs `Order(item="Widget", quantity=10, unit_price=10.0)` with the same boilerplate, customising only the one field relevant to that test.

**The solution — a factory fixture that accepts overrides:**

```python
# tests/conftest.py — factory pattern using a closure

from __future__ import annotations
from typing import Any
import pytest
from src.orders import Order

@pytest.fixture
def make_order():
    """
    Factory fixture. Returns a callable that creates Order objects.
    Callers override only what they care about.
    Defaults produce a valid, non-trivially-small order.
    """
    def _factory(
        item: str = "Widget",
        quantity: int = 5,
        unit_price: float = 10.0,
        **kwargs: Any,
    ) -> Order:
        return Order(item=item, quantity=quantity, unit_price=unit_price)

    return _factory
```

```python
# tests/test_orders.py — clean tests using the factory

def test_total_no_discount(make_order):
    order = make_order()                     # Default order, quantity=5
    assert calculate_total(order) == 50.0


def test_bulk_discount_threshold(make_order):
    order = make_order(quantity=10)          # Only override what matters
    assert apply_bulk_discount(order) == 85.0


def test_high_value_item(make_order):
    order = make_order(unit_price=999.99, quantity=1)
    assert calculate_total(order) == pytest.approx(999.99)


def test_zero_price_item(make_order):
    order = make_order(unit_price=0.0)
    assert calculate_total(order) == 0.0

# Notice: each test expresses ONLY the variation relevant to it.
# The factory supplies sane defaults for everything else.
```

```
┌─────────────────────────────────────────────────────────────────┐
│               FACTORY FIXTURE vs factory_boy                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The pattern above (closure returning a callable) is enough     │
│  for most projects. No extra dependency.                        │
│                                                                 │
│  factory_boy is a dedicated library for this pattern:           │
│  ├─ class OrderFactory(factory.Factory):                        │
│  │       class Meta: model = Order                              │
│  │       item = "Widget"                                        │
│  │       quantity = factory.Sequence(lambda n: n + 1)           │
│  │       unit_price = 10.0                                      │
│  │                                                              │
│  ├─ Use it when: objects are complex (many fields, relations)   │
│  └─ Especially useful with SQLAlchemy models (Week 6)           │
│                                                                 │
│  For now: the closure pattern above is sufficient.              │
│  factory_boy will be introduced properly in Week 6.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.6 Test Parallelization

**Running tests in parallel with `pytest-xdist`:**

```bash
$ uv add --dev pytest-xdist

# Run tests across all CPU cores automatically:
$ pytest -n auto

# Run on a fixed number of workers:
$ pytest -n 4

# Example output:
# 4 workers [120 items]
# .... [  3%]
# ........ [ 10%]
# 1 failed, 119 passed in 4.3s    ← 4x faster than sequential
```

```
┌─────────────────────────────────────────────────────────────────┐
│             PARALLELIZATION — WHEN IT HELPS AND HURTS           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HELPS:                                                         │
│  ├─ Large suites with many independent tests                    │
│  └─ Tests with real I/O (file system, network, DB)              │
│                                                                 │
│  HURTS:                                                         │
│  ├─ Tests that share mutable global state (module-level vars)   │
│  ├─ Tests that all write to the same database without           │
│  │   transaction isolation                                       │
│  └─ Tests that depend on execution ORDER                        │
│                                                                 │
│  RULE: If your tests are truly isolated (no shared state),      │
│  -n auto is safe and free speed. If tests fail only under -n,   │
│  you have a shared-state bug — find and fix it.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.7 Test Reporting

**Generating reports for teams and CI systems:**

```bash
# HTML report (visual, shareable with team):
$ uv add --dev pytest-html
$ pytest --html=report.html --self-contained-html
# Opens in browser. Shows pass/fail/skip per test, timestamps, durations.

# JUnit XML (for CI systems: GitHub Actions, GitLab CI, Jenkins):
$ pytest --junitxml=junit.xml
# GitHub Actions reads this format natively.
# Displays individual test results in the Actions UI.

# Both at once:
$ pytest --html=report.html --self-contained-html --junitxml=junit.xml
```

```toml
# Add JUnit XML to addopts for automatic CI reporting:
[tool.pytest.ini_options]
addopts = """
    --tb=short
    -v
    --strict-markers
    --cov=src
    --cov-report=term-missing
    --cov-fail-under=80
    --junitxml=junit.xml
"""
```

---

## 8.8 CI/CD Integration Preview

> "Full CI/CD pipeline setup is covered in Week 15. Here is the minimal GitHub Actions workflow to run your tests automatically on every push — so you know this is where everything you've learned today plugs in."

```yaml
# .github/workflows/tests.yml

name: Tests

on:
  push:
    branches: ["main", "develop"]
  pull_request:
    branches: ["main"]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v3

      - name: Install dependencies
        run: uv sync --all-extras

      - name: Run tests with coverage
        run: uv run pytest
        # Picks up all flags from [tool.pytest.ini_options] in pyproject.toml
        # --cov-fail-under=80 → step fails if coverage drops below 80%

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        if: always()    # Upload even if tests fail
        with:
          name: coverage-report
          path: htmlcov/

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: junit.xml
```

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT HAPPENS ON EACH PUSH                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Developer pushes code                                          │
│       │                                                         │
│       ▼                                                         │
│  GitHub Actions starts the workflow                             │
│       │                                                         │
│       ▼                                                         │
│  uv sync installs exact versions from uv.lock                   │
│  (Same environment on every machine, every time)                │
│       │                                                         │
│       ▼                                                         │
│  pytest runs — same flags as local dev (pyproject.toml)         │
│       │                                                         │
│       ├── All pass, coverage ≥ 80% → ✅ PR can be merged        │
│       │                                                         │
│       └── Any failure OR coverage < 80% → ❌ PR is blocked      │
│           Developer must fix before merge                       │
│                                                                 │
│  "It works on my machine" dies here.                            │
│  Tests are now the objective source of truth.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│        ✏️  PRACTICE CHECKPOINT — PART 8 (8.1 – 8.2)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  These two sections are directly needed for the mini-project.   │
│                                                                 │
│  1. Your CLI has a `fetch` command that takes a --city argument  │
│     and prints "Fetching <city>...". Write the full test using   │
│     CliRunner. What does result.exit_code == 0 verify?           │
│                                                                 │
│  2. Your weather command fails silently if the API key is not   │
│     set. Write a test that:                                     │
│     a) ensures WEATHER_API_KEY is absent                        │
│     b) invokes the CLI                                          │
│     c) asserts the exit code is non-zero and the output         │
│        contains "API key not set"                               │
│                                                                 │
│  3. You have a function last_updated() that returns a string    │
│     like "Updated at 14:32 on 2024-06-15". Write a freeze_time  │
│     test that asserts the exact string.                         │
│                                                                 │
│  4. A test fails ONLY when run with pytest -n auto but passes   │
│     alone. What type of problem does this indicate?             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solutions:**

```python
# ── Answer 1 ──────────────────────────────────────────────────────
from click.testing import CliRunner
from src.main import cli

def test_fetch_command_outputs_city(runner: CliRunner):
    result = runner.invoke(cli, ["fetch", "--city", "Paris"])

    assert result.exit_code == 0
    # exit_code == 0 verifies: the command completed without calling
    # sys.exit() with a non-zero code and no unhandled exception was raised.
    assert "Fetching Paris" in result.output


# ── Answer 2 ──────────────────────────────────────────────────────
def test_missing_api_key_shows_error(
    runner: CliRunner,
    monkeypatch: pytest.MonkeyPatch,
):
    monkeypatch.delenv("WEATHER_API_KEY", raising=False)

    result = runner.invoke(cli, ["weather", "London"])

    assert result.exit_code != 0
    assert "API key not set" in result.output


# ── Answer 3 ──────────────────────────────────────────────────────
# src/reporter.py (assumed implementation):
# from datetime import datetime
# def last_updated() -> str:
#     now = datetime.now()
#     return f"Updated at {now:%H:%M} on {now:%Y-%m-%d}"

from freezegun import freeze_time
from src.reporter import last_updated

@freeze_time("2024-06-15 14:32:00")
def test_last_updated_format():
    result = last_updated()
    assert result == "Updated at 14:32 on 2024-06-15"
# Without freeze_time, this assertion would fail 59 out of 60 minutes.


# ── Answer 4 ──────────────────────────────────────────────────────
# A shared state / test isolation problem.
# Tests running in parallel are interfering with each other — likely
# through: a shared module-level variable, a file both tests write to,
# or test ordering dependencies.
# The fix: ensure each test creates its own isolated resources
# (use tmp_path, make_order factory, function-scoped fixtures).
# NEVER use module-level mutable state in tests.
```

---

# QUICK REFERENCE CARD

> ADD the following sections to the existing Quick Reference Card:

```
┌─────────────────────────────────────────────────────────────────┐
│           PYTEST QUICK REFERENCE  (additions)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SKIP / XFAIL:                                                  │
│      @pytest.mark.skip(reason="...")                            │
│      @pytest.mark.skipif(condition, reason="...")               │
│      @pytest.mark.xfail(reason="...", strict=True)              │
│      pytest.skip("msg")   ← skip from inside test body         │
│                                                                 │
│  CUSTOM MARKERS:                                                │
│      @pytest.mark.slow                                          │
│      @pytest.mark.integration                                   │
│      pytest -m "not slow"        ← filter by marker            │
│      pytest -m "smoke or unit"                                  │
│      Register in pyproject.toml: markers = ["slow: ..."]        │
│                                                                 │
│  ADVANCED PARAMETRIZE:                                          │
│      pytest.param(val, marks=pytest.mark.xfail(...))            │
│      @pytest.mark.parametrize("a,b", [...], ids=["name1", ...]) │
│                                                                 │
│  DEBUGGING FLAGS:                                               │
│      pytest --pdb         ← debugger on failure                 │
│      pytest --trace       ← debugger at start of each test      │
│      pytest -l            ← show local variables on failure     │
│      pytest --lf          ← run last-failed tests only          │
│      pytest --ff          ← run failed tests first              │
│      pytest -s            ← required for breakpoint() in tests  │
│                                                                 │
│  BUILT-IN FIXTURES:                                             │
│      tmp_path: Path           ← fresh temp dir per test         │
│      monkeypatch.setenv(k,v)  ← safe env var override          │
│      monkeypatch.delenv(k)    ← safe env var removal            │
│      monkeypatch.setattr(mod, "name", val)                      │
│      capsys.readouterr()      ← capture stdout/stderr           │
│      caplog.at_level(logging.INFO)                              │
│      caplog.text, caplog.records                                │
│                                                                 │
│  COVERAGE:                                                      │
│      pytest --cov=src --cov-report=term-missing                 │
│      pytest --cov=src --cov-branch                              │
│      pytest --cov=src --cov-report=html                         │
│      pytest --cov-fail-under=80                                 │
│                                                                 │
│  WARNINGS:                                                      │
│      with pytest.warns(DeprecationWarning, match="..."):        │
│          code_that_warns()                                      │
│                                                                 │
│  CLI TESTING:                                                   │
│      from click.testing import CliRunner                        │
│      runner = CliRunner()                                       │
│      result = runner.invoke(cli, ["command", "--flag", "val"])  │
│      result.exit_code, result.output, result.exception          │
│                                                                 │
│  TIME TESTING:                                                  │
│      from freezegun import freeze_time                          │
│      @freeze_time("2024-01-15 12:00:00")                        │
│      with freeze_time("2024-01-15 12:00:00"):                   │
│                                                                 │
│  REPORTING:                                                     │
│      pytest --junitxml=junit.xml                                │
│      pytest --html=report.html --self-contained-html            │
│                                                                 │
│  PARALLELIZATION:                                               │
│      pytest -n auto      ← all CPU cores (pytest-xdist)         │
│      pytest -n 4         ← fixed 4 workers                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Upcoming Lectures

> ADD the following entry to the existing "Connection to Upcoming Lectures" box:

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS (additions)                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 2 MINI-PROJECT (immediately):                             │
│  └─ Async CLI Weather/News Aggregator                           │
│     Your CLI is tested with CliRunner (Part 8.1).               │
│     freeze_time pins your "last updated" timestamps (Part 8.2). │
│     monkeypatch.setenv provides fake API keys (Part 3.6).       │
│     pytest-cov confirms 80%+ coverage before submission.        │
│     pytest -m "not slow" separates unit from integration tests. │
│                                                                 │
│  WEEK 6 (SQLAlchemy / Databases):                               │
│  └─ factory_boy replaces the manual factory pattern (8.5)       │
│     Yield fixtures for database transaction rollback (3.4)       │
│     caplog verifies your ORM query logging                      │
│                                                                 │
│  WEEK 15 (CI/CD):                                               │
│  └─ The GitHub Actions preview (8.8) becomes a full pipeline.   │
│     Branch protection rules enforce --cov-fail-under=80.        │
│     JUnit XML feeds test results into the Actions dashboard.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
