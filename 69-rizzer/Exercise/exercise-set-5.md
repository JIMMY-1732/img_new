# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "Debugging is scattering print statements until I find it. If the program runs without crashing, it works. The IDE debugger is just a fancier print statement tool. Catching every exception means the program is robust. Leaving a `breakpoint()` in is harmless until I need to re-investigate."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---|---|
| 1 | Read tracebacks top-to-bottom; the first frame is the problem | The LAST LINE (error + message) and the BOTTOM FRAME in your code are where you start; the top frame is just the call entry point |
| 2 | Print statements are the natural first step after a crash | The traceback already contains the answer; reaching for print() is ignoring the report on the crime that's already been filed |
| 3 | The IDE debugger is a visual print statement — it shows variables | The debugger freezes live execution, lets you navigate the entire call stack, inspect all scope simultaneously, and test hypotheses interactively |
| 4 | "Step Into" on every line is the most thorough way to use the debugger | Stepping into every line descends into Python stdlib internals and library code; your hypothesis determines where you step |
| 5 | A program that runs without crashing is correct | Wrong output, performance bugs, race conditions, and memory leaks are all bugs that produce no crash and no traceback |
| 6 | `except Exception` is defensive coding | It swallows programming errors (typos, wrong types, wrong keys) alongside expected failures, making bugs invisible and unreproducible |
| 7 | `breakpoint()` left in code is harmless — it only activates when you need it | `breakpoint()` is not a comment. Every execution reaches that line and PAUSES the process, waiting for terminal input that never arrives in production |
| 8 | Logging is just verbose printing; `print()` works the same way | Logging has severity levels filterable at runtime without code changes, survives in remote/containerized environments, and is the ONLY tool that works on intermittent production failures |
| 9 | `assert` and `raise ValueError` are interchangeable | `assert` can be disabled at runtime with `python -O` and signals programmer-assumption violations; `raise` is always active and signals domain/data errors the caller should handle |

---

## The Main Purpose (Essence)

**Core Insight:** Every bug is a gap between what you *think* your code does and what it *actually* does. Every debugging technique in this lecture is a different instrument for closing that gap. The skill is not knowing how to operate the instruments — it is knowing which instrument to reach for given the TYPE of gap you are facing. The wrong instrument makes the gap wider.

**The Problems Being Solved:**

| Technique | Without It | With It |
|---|---|---|
| Bottom-up traceback reading | 5 minutes guessing which function to look at | Crash site identified in 5 seconds |
| IDE Debugger | Add/remove/forget print statements in a loop | See ALL scope simultaneously; navigate live execution |
| pdb | No debugging on SSH servers or containers | Full debugging capability everywhere Python runs |
| Logging | Production failures are invisible | Every failure leaves evidence; intermittent bugs become diagnosable |
| Hypothesis-driven debugging | Step through 400 lines looking for "something wrong" | Each debugging action tests a specific theory; converge in minutes |
| Assertions | Confusing crash 3 functions from the actual assumption violation | Failure happens AT the violated assumption, with your message naming what was wrong |

**The Key Distinctions:**
- **assert vs raise**: `assert` is a programmer's self-check, disabled in optimized mode; `raise` enforces domain rules, always active
- **print vs logging**: `print` is a disposable scalpel; `logging` is a permanent, level-controlled instrument
- **"where it crashed" vs "the root cause"**: These are often different frames in the same traceback — the crash site is where execution stopped; the root cause is where wrong data entered

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "Read traceback top-to-bottom" | "The bug is in the top-level function — add try/except there" | Crash is in the bottom frame; a fix there hides the problem instead of solving it | Traceback is printed oldest-to-newest; the error is always the last line |
| "Program runs = program works" | "No crash = correct output" | Silent bugs produce confidently wrong output with no exception raised | Crashes are only one class of bug; wrong output, wrong performance, wrong memory use are all bugs |
| "Step Into on every line" | "I'll see everything and find the bug" | Descends into `list.__contains__`, socket internals, Python stdlib | Hypothesis must guide direction; step INTO only when the bug is suspected inside that function |
| "`except Exception` is safe" | "Any unexpected failure is handled gracefully" | A typo `"emial"` instead of `"email"` causes `KeyError`, caught silently, returns "user not found" for a valid user | `Exception` catches ALL exceptions including programmer errors |
| "`breakpoint()` is harmless" | "Production still works; breakpoint just waits to be triggered" | Every request that hits that line freezes indefinitely | `breakpoint()` executes and halts the process; it is not a conditional comment |
| "Debugger finds all bugs" | "Attach debugger to reproduce the race condition" | Race condition disappears when debugger is attached | Debugger changes timing; bugs that depend on concurrent interleaving cannot be observed interactively |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Inverted Reading

Read the code and traceback below. **Do not run it.** Answer all questions using only the traceback text and the source code.

```python
# catalog_importer.py
from dataclasses import dataclass

@dataclass
class LibraryBook:
    title: str
    author: str
    pages: int
    isbn: str

def normalize_pages(raw_value) -> int:
    """Convert a raw page count value to an integer."""
    return int(raw_value)

def create_book(record: dict) -> LibraryBook:
    return LibraryBook(
        title=record["title"],
        author=record["author"],
        pages=normalize_pages(record["pages"]),
        isbn=record["isbn"],
    )

def filter_long_reads(books: list[LibraryBook], min_pages: int) -> list[LibraryBook]:
    return [b for b in books if b.pages >= min_pages]

def process_import(raw_records: list[dict], min_pages: int = 200) -> list[LibraryBook]:
    catalog = [create_book(r) for r in raw_records]
    return filter_long_reads(catalog, min_pages)

raw_data = [
    {"title": "Clean Code",               "author": "Martin",  "pages": 464,   "isbn": "978-0132350884"},
    {"title": "Design Patterns",          "author": "GoF",     "pages": "395", "isbn": "978-0201633610"},
    {"title": "The Pragmatic Programmer", "author": "Hunt",    "pages": "TBD", "isbn": "978-0135957059"},
    {"title": "SICP",                     "author": "Abelson", "pages": 657,   "isbn": "978-0262510875"},
]

result = process_import(raw_data)
print(f"Found {len(result)} long reads")
```

**The traceback this program produces:**

```
Traceback (most recent call last):
  File "catalog_importer.py", line 37, in <module>
    result = process_import(raw_data)
  File "catalog_importer.py", line 27, in process_import
    catalog = [create_book(r) for r in raw_records]
  File "catalog_importer.py", line 27, in <listcomp>
    catalog = [create_book(r) for r in raw_records]
  File "catalog_importer.py", line 19, in create_book
    pages=normalize_pages(record["pages"]),
  File "catalog_importer.py", line 12, in normalize_pages
    return int(raw_value)
ValueError: invalid literal for int() with base 10: 'TBD'
```

**Questions:**

1. **The Wrong Approach.** A student reads the traceback top-to-bottom, sees `process_import` first at line 27, and proposes this fix:

   ```python
   def process_import(raw_records: list[dict], min_pages: int = 200) -> list[LibraryBook]:
       try:
           catalog = [create_book(r) for r in raw_records]
       except ValueError:
           catalog = []
       return filter_long_reads(catalog, min_pages)
   ```

   Identify everything wrong with this fix. Does it actually resolve the bug? What does it do to the program's behavior in production? What specific information in the traceback did the student fail to process?

2. **The Correct Reading.** Apply the three-step bottom-up technique:
   - **Step 1:** What does the LAST LINE tell you? What specific value caused the crash?
   - **Step 2:** What does the BOTTOM FRAME tell you? What question does it immediately raise that you can answer from the error message alone?
   - **Step 3:** Reading upward from `normalize_pages`: trace exactly how the string `'TBD'` arrived at `int(raw_value)`, naming each frame.

3. **Without touching the code:** Using only the traceback text, identify the specific record that caused the crash. Explain your reasoning precisely.

4. **Two fix locations.** Describe two defensible places to apply the fix, with the code for each. Then explain which one the lecture's principle — *"the bottom frame gives you the crash site; reading upward gives you the context"* — would lead you toward, and why.

---

## Exercise 1.2: The Successful Failure

This program runs to completion with no traceback. It is wrong. Your task is to diagnose the bug without running the code and without any error message to guide you.

```python
# margin_analyzer.py

def compute_margin(revenue: float, cost: float) -> float:
    """
    Returns gross margin as a ratio between 0.0 and 1.0.
    Example: revenue=100, cost=60 → 0.40 (which represents 40% margin)
    """
    return (revenue - cost) / revenue

def classify_product(name: str, revenue: float, cost: float) -> dict:
    margin = compute_margin(revenue, cost)

    # Intended thresholds: below 20% = low, 20–40% = medium, above 40% = high
    if margin > 40:
        category = "high"
    elif margin > 20:
        category = "medium"
    else:
        category = "low"

    return {"name": name, "margin_ratio": margin, "category": category}

products = [
    ("Premium Widget",  500.0, 150.0),  # margin: 70%
    ("Standard Widget", 200.0, 130.0),  # margin: 35%
    ("Budget Widget",   100.0,  90.0),  # margin: 10%
]

for args in products:
    result = classify_product(*args)
    print(f"{result['name']}: {result['margin_ratio']:.1%} margin → {result['category']}")
```

**Questions:**

1. **Predict the exact output** for all three products. Write each line.

2. **Identify the gap.** The comments in the code say "margin: 70%", "margin: 35%", "margin: 10%". Based on the stated thresholds, what output did the developer *intend*? Where does your prediction in Question 1 differ from the intent?

3. **Form a hypothesis.** Without running the code, state in one sentence what you believe is the root cause. Use the format: *"My hypothesis is that ____."*

4. **Test the hypothesis with the debugger.** Describe precisely:
   - Which function you would place a breakpoint in, and on which line
   - What single expression you would add to the Watch pane to confirm or refute your hypothesis in one step
   - What value that Watch expression would show, and what it would prove

5. **Fix** the bug with the minimum possible change. Show exactly which lines change. Then mentally trace the fixed code for Premium Widget to verify.

6. **Prevention.** Write a single `assert` statement inside `classify_product`, placed after `margin = compute_margin(...)`, that would catch this class of unit-confusion bug by verifying the margin value is within the range the thresholds assume.

---

## Exercise 1.3: Tool Selection Under Pressure

For each scenario below, name the PRIMARY debugging tool. Then explain in one sentence why each of the OTHER two tools in the three-tool model (print statement / IDE debugger / logging) would be **inferior** for this specific scenario. Finally, state the first concrete action you would take with your chosen tool.

**Scenario A:** You are working locally in VS Code. A FastAPI endpoint `POST /invoices` crashes with `AttributeError: 'NoneType' object has no attribute 'customer_id'`. You can reproduce it every time by sending the same JSON body. The endpoint works for all other inputs you have tried.

**Scenario B:** A nightly data-sync job runs on a production Linux server accessible only via SSH. For the past four nights, it has silently returned between 847 and 1,000 of the expected 1,000 records, with no pattern you can identify. You cannot reproduce the inconsistency on your local machine.

**Scenario C:** You wrote a 10-line function that computes a rolling 3-day average from a list of readings. Your unit test passes for a 3-item list but fails for a 7-item list. You want to watch exactly which loop iteration first produces the wrong value.

**Scenario D:** A request handler that was taking 60ms is now taking 4,800ms after you merged a new feature. You need to identify which function call is responsible before the end-of-day standup.

**Scenario E:** You want to verify that a helper function `parse_iso_date("2024-01-15")` returns a `datetime.date` (not `datetime.datetime`) before writing the formal unit test. The check will take 10 seconds.

---

# Level 1 Solutions

## Solution 1.1

**1. Everything wrong with the top-down fix:**

The proposed fix catches `ValueError` and silently returns an empty catalog. This has three distinct problems:

- **It does not fix the bug.** The bad data (`"pages": "TBD"`) still exists. The fix hides the fact that one record is malformed.
- **It makes behavior worse.** Any future record with invalid page data — a blank field, a stray newline, any non-numeric string — will silently produce an empty library. The function will appear to succeed (no exception, returns a list) while delivering zero results. The caller cannot distinguish "nothing matched the filter" from "the entire import was invalid."
- **It discards the critical evidence.** The last line of the traceback says `ValueError: invalid literal for int() with base 10: 'TBD'`. The bad value, `'TBD'`, is printed verbatim in the error message. A student who read bottom-up had the complete answer in five seconds and needed no try/except at all.

**2. The three-step bottom-up reading:**

- **Step 1 (Last Line):** `ValueError: invalid literal for int() with base 10: 'TBD'`. Python's built-in `int()` received the string `'TBD'` and cannot parse it. Something in the data contains the string `'TBD'` where an integer is expected.

- **Step 2 (Bottom Frame):** `File "catalog_importer.py", line 12, in normalize_pages` — `return int(raw_value)`. The crash is inside `normalize_pages`, on the `int()` call. Immediate question: "What is `raw_value`?" The error message already answers: `'TBD'`.

- **Step 3 (Reading upward):** One frame up: `create_book` at line 19 — `pages=normalize_pages(record["pages"])`. So `record["pages"]` is `'TBD'`. This means a record dict has the string `'TBD'` for its `pages` key. One frame up: `process_import` is building the list by calling `create_book(r)` for each record in `raw_records`. This is the pipeline. The malformed record is in the raw input.

**3. Which record:**

The third record: `{"title": "The Pragmatic Programmer", "author": "Hunt", "pages": "TBD", "isbn": "978-0135957059"}`.

**Evidence:** The error message contains the exact bad value — `'TBD'`. Scanning the raw_data list, only one record has `"TBD"` for its `pages` field. No running required. The error message is the evidence.

**4. Two fix locations:**

**Fix A — In `normalize_pages` (the crash site):**
```python
def normalize_pages(raw_value) -> int:
    try:
        return int(raw_value)
    except ValueError:
        raise ValueError(
            f"Page count '{raw_value}' is not a valid integer."
        ) from None
```
This gives a more descriptive error, but it still doesn't tell you *which book* failed.

**Fix B — In `create_book` (the usage site — superior):**
```python
def create_book(record: dict) -> LibraryBook:
    raw_pages = record["pages"]
    if not str(raw_pages).lstrip("-").isdigit():
        raise ValueError(
            f"Book '{record['title']}' has an invalid page count: '{raw_pages}'. "
            "Expected an integer."
        )
    return LibraryBook(
        title=record["title"],
        author=record["author"],
        pages=int(raw_pages),
        isbn=record["isbn"],
    )
```

**Why Fix B is better:** Reading one frame up from the crash site puts you in `create_book`, which has both the bad value AND the book's context (title, isbn). The error message from Fix B names the specific book. The lecture's principle — "error + bottom frame = 80% of your answer, then read upward" — leads you to `create_book` because that is the frame with enough context to write a useful error message.

---

## Solution 1.2

**1. Predicted output:**
```
Premium Widget: 70.0% margin → low
Standard Widget: 35.0% margin → low
Budget Widget: 10.0% margin → low
```

**2. The gap:** The developer intended "high" for Premium, "medium" for Standard, "low" for Budget. All three print "low" instead.

**3. The hypothesis:** "My hypothesis is that `compute_margin` returns a ratio in the range 0.0–1.0 (e.g., `0.70` for 70% margin), but the classification thresholds compare it to whole numbers (`40` and `20`), so the condition `0.70 > 40` is always False, causing every product to fall through to `'low'`."

**4. Testing with the debugger:**
- **Breakpoint:** Line `if margin > 40:` inside `classify_product`, immediately after `margin = compute_margin(revenue, cost)`.
- **Watch expression:** `margin` (and optionally `margin > 40`).
- **What it shows:** For Premium Widget, `margin = 0.7`. The Watch pane will show `margin > 40` evaluates to `False`. This confirms the hypothesis: `0.7` is not greater than `40`, so "high" is never reached.

**5. Fix — change thresholds from whole numbers to ratios:**
```python
# Change:
if margin > 40:
    category = "high"
elif margin > 20:
    category = "medium"

# To:
if margin > 0.40:
    category = "high"
elif margin > 0.20:
    category = "medium"
```

Mental trace for Premium Widget: `compute_margin(500.0, 150.0)` = `(500-150)/500` = `350/500` = `0.70`. `0.70 > 0.40` → True → `"high"` ✓

**6. Prevention assertion:**
```python
def classify_product(name: str, revenue: float, cost: float) -> dict:
    margin = compute_margin(revenue, cost)
    assert 0.0 <= margin <= 1.0, (
        f"Expected margin as a ratio in [0.0, 1.0], got {margin!r}. "
        f"If this is a percentage (0–100), divide by 100 first."
    )
    ...
```

This fires the moment a caller mistakenly pre-computes a percentage and passes it in, catching the unit confusion at the boundary instead of silently misclassifying every product.

---

## Solution 1.3

**Scenario A — local, reproducible, GUI available → IDE Debugger**
- **Why not print:** Shows one variable at a time; you'd need multiple run/edit cycles to see all the state involved in `'NoneType' object has no attribute 'customer_id'`. The debugger shows every local variable at the crash frame simultaneously.
- **Why not logging:** Adds code that must be cleaned up later; logging is the right tool for non-interactive environments, not for a local reproducible crash where you can pause and inspect in real time.
- **First action:** Set a conditional breakpoint where the `AttributeError` occurs. Use the Call Stack pane to click the frame just above it — inspect the variable that is `None` and identify which function returned `None` when it should have returned an object.

**Scenario B — remote, intermittent, no GUI → Logging**
- **Why not IDE debugger:** No GUI; and attaching a debugger to a nightly process would require restarting it and may not reproduce the intermittent failure, which by definition requires specific runtime conditions.
- **Why not print:** In a daemonized server process, stdout may be discarded or interleaved incoherently. Print statements also cannot be filtered by severity and leave cleanup debt.
- **First action:** Add `logger.debug()` calls at every point where a record is processed or skipped. Deploy. Run the job. Read the log the next morning — the debug output will show exactly which records were processed and which were skipped and why.

**Scenario C — local, specific iteration failure, needs step-through → IDE Debugger**
- **Why not print:** Requires editing the loop, adding multiple prints, re-running, reading mixed output — many steps when you could pause at exactly the failing iteration interactively.
- **Why not logging:** Logging is the non-interactive alternative; here you have a GUI and can watch variables update live at each step, which is strictly superior.
- **First action:** Set a conditional breakpoint inside the loop at the line where the rolling average is computed, with a condition that triggers only when the iteration index is 3 or higher (where the test first fails). Add `window` and `avg` to the Watch pane. Step through one iteration at a time.

**Scenario D — performance bug → `cProfile` (performance tooling, not the three-tool model)**
- **Among the three-tool model:** Logging with `time.perf_counter()` around each major step is the closest fit.
- **Why not IDE debugger:** The debugger adds overhead to every bytecode operation. Any timing measurement made inside a debugger session is contaminated and meaningless for performance diagnosis.
- **Why not print:** A raw print gives elapsed time for one specific location; `cProfile` gives a complete breakdown of every function call and is sorted by time consumed.
- **First action:** Wrap the request handler in `cProfile.Profile()`, trigger the slow request, inspect `pstats.Stats` sorted by `CUMULATIVE`. The function with the highest `tottime` that is not a wrapper is the bottleneck.

**Scenario E — quick local throwaway check → Print statement**
- **Why not IDE debugger:** Launching the full debugger, setting a breakpoint, stepping through, reading the Variables pane — this is two minutes of overhead for a ten-second check.
- **Why not logging:** Requires `basicConfig`, `getLogger`, choosing a level — entirely disproportionate to a disposable one-time check.
- **First action:** `print(type(parse_iso_date("2024-01-15")))`. Delete immediately. Write the proper test.

---

# LEVEL 2: DEBUG, FIX & VERIFY

## Exercise 2.1: The Wrong Weapon

**Context:** A developer is debugging a shipping cost calculator. The system applies a 15% volume discount when ordering 10 or more units. An order for exactly 10 units isn't getting the discount. The developer tried print-statement debugging — and their output ALREADY CONTAINS the answer, but they missed it and kept adding more prints.

```python
# shipping_calculator.py
from dataclasses import dataclass

@dataclass
class OrderLine:
    product_id: str
    quantity: int
    unit_price: float

def compute_subtotal(line: OrderLine) -> float:
    return line.quantity * line.unit_price

def apply_volume_discount(subtotal: float, quantity: int) -> float:
    """Apply 15% discount when ordering 10 or more units."""
    if quantity > 10:       # ← examine this carefully
        return subtotal * 0.85
    return subtotal

def apply_shipping_tax(amount: float, rate: float = 0.08) -> float:
    return round(amount * (1 + rate), 2)

def calculate_order_total(line: OrderLine) -> float:
    subtotal   = compute_subtotal(line)
    discounted = apply_volume_discount(subtotal, line.quantity)
    total      = apply_shipping_tax(discounted)
    return total

# ===== DEVELOPER'S DEBUG ATTEMPT =====
# Order: 10 units of PROD-X at $50.00 each
# Expected: 10 × $50 = $500, 15% discount → $425, 8% tax → $459.00
# Actual:   calculate_order_total returns $540.00 — discount not applied

def calculate_order_total_debug(line: OrderLine) -> float:
    print(f"[D] line={line}")
    subtotal = compute_subtotal(line)
    print(f"[D] subtotal={subtotal}")
    discounted = apply_volume_discount(subtotal, line.quantity)
    print(f"[D] discounted={discounted}")          # ← the answer is visible here
    total = apply_shipping_tax(discounted)
    print(f"[D] after_tax={total}")
    return total

test_order = OrderLine("PROD-X", 10, 50.0)
result = calculate_order_total_debug(test_order)
print(f"\nResult:   {result}")
print(f"Expected: 459.00")
```

**The debug output:**
```
[D] line=OrderLine(product_id='PROD-X', quantity=10, unit_price=50.0)
[D] subtotal=500.0
[D] discounted=500.0
[D] after_tax=540.0

Result:   540.0
Expected: 459.00
```

---

**Task 2a: Identify the flaw in the debugging approach**

After seeing `discounted=500.0`, the developer opens `apply_volume_discount` and immediately adds a second round of prints inside it:

```python
def apply_volume_discount(subtotal: float, quantity: int) -> float:
    print(f"[D-INNER] subtotal={subtotal}, quantity={quantity}")
    print(f"[D-INNER] {quantity} > 10 evaluates to: {quantity > 10}")
    if quantity > 10:
        return subtotal * 0.85
    return subtotal
```

Without running any more code: what information did the developer **already have** in their first debug output that made this second round completely unnecessary? Name the specific debugging skill they failed to apply between the two rounds.

---

**Task 2b: Explain the mechanism**

Focus on `apply_volume_discount`. `quantity = 10`. The condition is `10 > 10`.

What does this evaluate to? Why? What does the comment "Apply 15% discount when ordering 10 or more units" tell you about the programmer's INTENT versus the code's BEHAVIOR?

---

**Task 2c: Instrument to prove diagnosis**

The developer's first output showed `[D] discounted=500.0`. Rather than adding more prints, a hypothesis-driven approach forms one testable prediction from this single data point.

Write the hypothesis in the lecture's format:

*"My hypothesis is that ____. If correct, calling `apply_volume_discount(500.0, 10)` directly will return ____, while `apply_volume_discount(500.0, 11)` will return ____."*

Show what the direct test reveals:
```python
print(apply_volume_discount(500.0, 10))   # ?
print(apply_volume_discount(500.0, 11))   # ?
```

---

**Task 2d: Fix the code**

Make the minimum possible change. Then mentally trace the fixed code for `OrderLine("PROD-X", 10, 50.0)`, showing each intermediate value through to the final total.

---

**Task 2e: Write verification tests**

Write two tests: one that **fails on the original** (broken) code and **passes on the fixed** code, and one that was already passing and must remain passing as a regression guard. Fill in the correct expected values and explain how the pair together proves the boundary condition is correct.

```python
def test_volume_discount_applies_at_exactly_10_units():
    """Boundary: 10 units must receive the 15% discount."""
    line = OrderLine("TEST", 10, 50.0)
    result = calculate_order_total(line)
    assert result == ???, f"Expected ???, got {result}"

def test_volume_discount_does_not_apply_at_9_units():
    """9 units must NOT receive the discount."""
    line = OrderLine("TEST", 9, 50.0)
    result = calculate_order_total(line)
    assert result == ???, f"Expected ???, got {result}"
```

---

## Exercise 2.2: The Invisible Guard

**Context:** A data analyst built an NPS scoring system. It crashes with a confusing, generic error when a survey has no responses — because the bad data enters the system unchecked and propagates three calls deep before anything notices.

```python
# survey_scorer.py
from dataclasses import dataclass, field

@dataclass
class Survey:
    survey_id: str
    title: str
    responses: list[int] = field(default_factory=list)

    def net_promoter_score(self) -> float:
        """
        NPS = ((promoters - detractors) / total) * 100
        Promoters:  responses of 9 or 10
        Detractors: responses of 6 or below
        """
        promoters  = sum(1 for r in self.responses if r >= 9)
        detractors = sum(1 for r in self.responses if r <= 6)
        total      = len(self.responses)
        return ((promoters - detractors) / total) * 100

def load_surveys(raw_data: list[dict]) -> list[Survey]:
    return [
        Survey(survey_id=r["id"], title=r["title"], responses=r["responses"])
        for r in raw_data
    ]

def rank_by_nps(surveys: list[Survey]) -> list[tuple[str, float]]:
    return sorted(
        [(s.survey_id, s.net_promoter_score()) for s in surveys],
        key=lambda x: x[1],
        reverse=True,
    )

raw = [
    {"id": "Q4-2024", "title": "Q4 Product Survey",     "responses": [9, 10, 7, 9, 6, 10]},
    {"id": "Q3-2024", "title": "Q3 Product Survey",     "responses": [6, 5, 8, 7, 6]},
    {"id": "PILOT",   "title": "Pilot (no responses)",  "responses": []},   # ← problem
    {"id": "Q2-2024", "title": "Q2 Product Survey",     "responses": [10, 9, 10, 8]},
]

surveys = load_surveys(raw)
rankings = rank_by_nps(surveys)
for survey_id, nps in rankings:
    print(f"{survey_id}: NPS = {nps:.1f}")
```

**The crash:**
```
Traceback (most recent call last):
  File "survey_scorer.py", line 31, in <module>
    rankings = rank_by_nps(surveys)
  File "survey_scorer.py", line 24, in rank_by_nps
    [(s.survey_id, s.net_promoter_score()) for s in surveys],
  File "survey_scorer.py", line 24, in <listcomp>
    [(s.survey_id, s.net_promoter_score()) for s in surveys],
  File "survey_scorer.py", line 15, in net_promoter_score
    return ((promoters - detractors) / total) * 100
ZeroDivisionError: float division by zero
```

---

**Task 2a: Identify the flaw**

A developer proposes this fix:

```python
def net_promoter_score(self) -> float:
    ...
    total = len(self.responses)
    if total == 0:
        return 0.0    # ← "fix"
    return ((promoters - detractors) / total) * 100
```

State the specific domain problem with returning `0.0` for an empty survey. What assumption does this silently encode, and why is it dangerous in a system that uses NPS scores to make product decisions?

---

**Task 2b: Explain the mechanism**

1. Which specific survey triggered the crash? How do you know from the traceback alone, without inspecting the data?
2. Why doesn't the current traceback tell you the survey's `id` or `title`? What would you need to add to the code so that the error message NAMES the problematic survey?

---

**Task 2c: Instrument to prove diagnosis**

Without modifying the NPS calculation logic, add a single `assert` statement inside `net_promoter_score` that:
- Guards against empty responses
- Produces an error message that names the specific survey

Show the output this assert would produce for the PILOT survey.

---

**Task 2d: Fix the code with proper exception handling**

The correct fix has two parts:

**Part 1:** In `net_promoter_score`, use `raise ValueError` (not `assert`) to signal that an empty survey is an invalid domain state. Justify in one sentence why `raise` is correct here and `assert` would be wrong for this specific scenario.

**Part 2:** Add a guard in `load_surveys` that rejects empty-response surveys at the point they ENTER the system — not by silently skipping them, but by raising a descriptive error that names the problematic survey.

---

**Task 2e: Write verification tests**

```python
import pytest

def test_empty_survey_raises_descriptive_error():
    """An empty survey must fail with a message that names the survey — not a ZeroDivisionError."""
    empty = Survey(survey_id="PILOT", title="Pilot", responses=[])
    with pytest.raises(ValueError) as exc_info:
        empty.net_promoter_score()
    assert "PILOT" in str(exc_info.value), (
        "Error message must name the problematic survey ID"
    )

def test_nps_computed_correctly_for_known_dataset():
    """Verify the NPS formula against a manually-calculated expected value."""
    # responses: [9, 10, 7, 9, 6, 10]
    # Promoters (≥9): 9, 10, 9, 10 → 4
    # Detractors (≤6): 6 → 1
    # NPS = ((4 - 1) / 6) * 100 = 50.0
    survey = Survey(survey_id="Q4", title="Q4", responses=[9, 10, 7, 9, 6, 10])
    assert survey.net_promoter_score() == 50.0
```

---

# Level 2 Solutions

## Solution 2.1

**Task 2a: The flaw in the debugging approach**

The first debug output already contained the complete answer:
```
[D] subtotal=500.0
[D] discounted=500.0   ← subtotal and discounted are equal
```

`discounted = 500.0` when `subtotal = 500.0` means `apply_volume_discount` returned its input unchanged — the discount was NOT applied. Combined with knowing `quantity=10` from the first debug line, you have every fact you need: the function received `quantity=10` and returned no discount.

The next step is not to add more prints. It is to **form a hypothesis from the existing evidence**: "The discount condition must not be triggering for `quantity=10`." Then look at the condition: `if quantity > 10`. Ask: "Is `10 > 10` True?" The answer is visible in the source code. The second round of prints gave the developer information they already had.

**The missed skill:** Hypothesis formation. The developer collected data but did not reason about it before collecting more. The lecture's principle is: form a testable hypothesis, then take the ONE action that confirms or refutes it.

---

**Task 2b: The mechanism**

`10 > 10` evaluates to `False`. Ten is not STRICTLY GREATER than ten. The comment says "10 or more units" — which means `>=` (greater-than-or-equal). The code uses `>` (strictly greater than). The discount requires 11 units by the code's behavior, not 10. This is a boundary condition off-by-one error: the most common class of discount and threshold logic bugs.

---

**Task 2c: Hypothesis and proof**

*"My hypothesis is that `apply_volume_discount` uses strict greater-than (`> 10`) instead of greater-than-or-equal (`>= 10`), so `quantity=10` does not trigger the 15% discount. If correct, calling `apply_volume_discount(500.0, 10)` will return `500.0` (no discount), while `apply_volume_discount(500.0, 11)` will return `425.0` (with discount)."*

```python
print(apply_volume_discount(500.0, 10))   # 500.0 — confirms hypothesis
print(apply_volume_discount(500.0, 11))   # 425.0 — confirms hypothesis
```

---

**Task 2d: Fix and trace**

Change `quantity > 10` to `quantity >= 10`.

Trace for `OrderLine("PROD-X", 10, 50.0)`:
- `compute_subtotal`: `10 × 50.0 = 500.0`
- `apply_volume_discount(500.0, 10)`: `10 >= 10` → True → `500.0 × 0.85 = 425.0`
- `apply_shipping_tax(425.0)`: `round(425.0 × 1.08, 2) = round(459.0, 2) = 459.0`
- Final: `459.0` ✓

---

**Task 2e: Verification tests**

```python
def test_volume_discount_applies_at_exactly_10_units():
    """Boundary: 10 units must receive the 15% discount."""
    line = OrderLine("TEST", 10, 50.0)
    result = calculate_order_total(line)
    # 10 × $50 = $500, 15% off = $425, 8% tax = $459.00
    assert result == 459.00, f"Expected 459.00, got {result}"

def test_volume_discount_does_not_apply_at_9_units():
    """9 units must NOT receive the discount."""
    line = OrderLine("TEST", 9, 50.0)
    result = calculate_order_total(line)
    # 9 × $50 = $450, no discount, 8% tax = $486.00
    assert result == 486.00, f"Expected 486.00, got {result}"
```

Together, these two tests prove the boundary is inclusive at exactly 10 units. The original code (`> 10`) would fail the first test because `10 > 10 = False` → no discount → result `540.00 ≠ 459.00`. It would pass the second (coincidentally, because 9 units also gets no discount under either `>` or `>=`). The pair together proves the boundary, not either test alone.

---

## Solution 2.2

**Task 2a: The domain problem**

An NPS of `0.0` is a real, meaningful score — it means promoters and detractors are exactly balanced. Returning `0.0` for an empty survey falsely implies the survey collected balanced feedback. In a system that ranks surveys by NPS, the PILOT survey would appear in the middle of the ranking, mixed with legitimate data. A business decision-maker might review dashboard results that include "PILOT: NPS = 0.0" and draw product conclusions from a survey that collected zero data points. **A silent wrong value is more dangerous than a crash** — crashes get reported and fixed; wrong values flow into decisions undetected.

---

**Task 2b: The mechanism**

1. The PILOT survey triggered the crash. Every survey in the list is processed inside `rank_by_nps`. The PILOT survey has `responses=[]`, so `len(self.responses) = 0`, causing the division by zero. This is the ONLY survey in the dataset with an empty responses list. That information is in the raw data, not in the traceback itself — the traceback only shows you the type of failure (`ZeroDivisionError`) and the line.

2. The traceback shows the call chain and the crash line, but not the VALUE of `self` at the crash point. To see `self.survey_id` in the error, you need to add it explicitly — either through an `assert`/`raise` that references `self.survey_id`, or by using `pdb.pm()` after the crash and running `p self` to inspect the object at the crash frame.

---

**Task 2c: Assert with survey name**

```python
def net_promoter_score(self) -> float:
    assert len(self.responses) > 0, (
        f"Survey '{self.survey_id}' has no responses. "
        "Cannot compute NPS on an empty dataset."
    )
    promoters  = sum(1 for r in self.responses if r >= 9)
    detractors = sum(1 for r in self.responses if r <= 6)
    total      = len(self.responses)
    return ((promoters - detractors) / total) * 100
```

For the PILOT survey, this produces:
```
AssertionError: Survey 'PILOT' has no responses. Cannot compute NPS on an empty dataset.
```

Instead of the opaque `ZeroDivisionError: float division by zero`.

---

**Task 2d: Proper exception handling**

**Part 1 — raise ValueError:**
```python
def net_promoter_score(self) -> float:
    if not self.responses:
        raise ValueError(
            f"Survey '{self.survey_id}' has no responses. "
            "NPS is undefined for surveys with no data."
        )
    promoters  = sum(1 for r in self.responses if r >= 9)
    detractors = sum(1 for r in self.responses if r <= 6)
    total      = len(self.responses)
    return ((promoters - detractors) / total) * 100
```

Why `raise` not `assert`: An empty survey is a foreseeable data condition from external input. The caller — `rank_by_nps`, or any future caller — might need to handle it gracefully (skip it, log a warning, display "N/A"). Using `raise ValueError` gives callers the option to `except ValueError` and respond appropriately. Using `assert` would be disabled by `python -O` and signals a programmer error, not a data condition.

**Part 2 — Guard at load time:**
```python
def load_surveys(raw_data: list[dict]) -> list[Survey]:
    surveys = []
    for r in raw_data:
        if not r["responses"]:
            raise ValueError(
                f"Survey '{r['id']}' ('{r['title']}') "
                "was loaded with no responses. Provide responses or remove this record."
            )
        surveys.append(Survey(
            survey_id=r["id"],
            title=r["title"],
            responses=r["responses"]
        ))
    return surveys
```

This fails at the ENTRY POINT — the data loading stage — with the survey ID visible in the message, before any computation begins. This is the precondition guard: catch bad data as early as possible.

---

**Task 2e: Tests**

```python
import pytest

def test_empty_survey_raises_descriptive_error():
    empty = Survey(survey_id="PILOT", title="Pilot", responses=[])
    with pytest.raises(ValueError) as exc_info:
        empty.net_promoter_score()
    assert "PILOT" in str(exc_info.value), (
        "Error message must name the survey ID so the caller can act on it"
    )

def test_nps_computed_correctly_for_known_dataset():
    # responses: [9, 10, 7, 9, 6, 10]
    # Promoters (≥9): 9, 10, 9, 10 → 4
    # Detractors (≤6): 6 → 1
    # NPS = ((4-1)/6) * 100 = 50.0
    survey = Survey(survey_id="Q4", title="Q4", responses=[9, 10, 7, 9, 6, 10])
    assert survey.net_promoter_score() == 50.0
```

---

# LEVEL 2.5: EXTEND EXISTING CODE

## Background

The following code is correct and working. It is a lightweight async service health checker using concepts from Lecture 3 (async/await, `asyncio.gather`). **You may not change any existing function signatures or return types. All three provided tests must remain green after each extension.**

```python
# health_checker.py
import asyncio
import logging
import time
from dataclasses import dataclass
from typing import Optional

logger = logging.getLogger(__name__)

@dataclass
class HealthResult:
    service_name: str
    status_code: int
    latency_ms: float
    error: Optional[str] = None

    @property
    def is_healthy(self) -> bool:
        return self.status_code == 200

async def check_service(name: str, url: str) -> HealthResult:
    """Simulate a health check. Replace with real httpx call in production."""
    start = time.perf_counter()
    await asyncio.sleep(0.05)   # simulated network latency
    elapsed_ms = (time.perf_counter() - start) * 1000

    if name.startswith("down-"):
        return HealthResult(name, 503, elapsed_ms, error="Service unavailable")
    return HealthResult(name, 200, elapsed_ms)

async def run_health_checks(services: dict[str, str]) -> list[HealthResult]:
    """Run all health checks concurrently."""
    tasks = [check_service(name, url) for name, url in services.items()]
    return await asyncio.gather(*tasks)

def build_summary(results: list[HealthResult]) -> dict:
    """Summarize health check results."""
    healthy = [r for r in results if r.is_healthy]
    return {
        "total":          len(results),
        "healthy":        len(healthy),
        "degraded":       len(results) - len(healthy),
        "avg_latency_ms": round(
            sum(r.latency_ms for r in healthy) / len(healthy), 2
        ) if healthy else 0.0,
    }
```

**Provided tests (must remain green after each extension):**

```python
import pytest

@pytest.mark.asyncio
async def test_all_healthy_services():
    services = {"api": "http://api.example.com", "db": "http://db.example.com"}
    results = await run_health_checks(services)
    summary = build_summary(results)
    assert summary["healthy"] == 2
    assert summary["degraded"] == 0
    assert summary["avg_latency_ms"] > 0

@pytest.mark.asyncio
async def test_degraded_service_detected():
    services = {"api": "http://api.example.com", "down-db": "http://db.example.com"}
    results = await run_health_checks(services)
    summary = build_summary(results)
    assert summary["healthy"] == 1
    assert summary["degraded"] == 1

def test_empty_healthy_returns_zero_latency():
    results = [HealthResult("down-svc", 503, 50.0, "unavailable")]
    summary = build_summary(results)
    assert summary["avg_latency_ms"] == 0.0
```

---

**Extension A — Structured Logging (Basic)**

Add `logging` calls to `check_service` so that:
- A successful check (status 200) emits `logger.info()` with the service name and latency in ms
- A failed check emits `logger.warning()` with the service name and error message
- No `print()` statements anywhere — use the module-level `logger` object

Show the log output you would expect (with `logging.basicConfig(level=logging.INFO)`) when checking `{"api": "http://...", "down-db": "http://..."}`.

---

**Extension B — Defensive Assertions (Intermediate)**

Add assertions to `build_summary` that guard against three classes of programmer error:

1. An empty `results` list — assert with a message describing what the caller should have done instead. **Constraint:** `test_empty_healthy_returns_zero_latency` must still pass — it sends a list with ONE result (an unhealthy service), which is not empty.

2. A negative `avg_latency_ms` in the return dict — assert with a message indicating a measurement bug.

3. The partitioning invariant: `healthy + degraded == total` — assert with a message showing all three values so a failure is immediately diagnosable.

---

**Extension C — Debug Scaffold for Intermittent Bug (Advanced)**

**Bug report:** "Once or twice per week, `avg_latency_ms` in the summary shows over 10,000 ms even though all services appear fine in other monitoring. We cannot reproduce it."

Without using the interactive debugger, add `logger.debug()` instrumentation to `build_summary` that captures — when the bug next occurs in production:

1. The individual `latency_ms` of every healthy result
2. The computed sum before division (to catch floating-point accumulation issues)
3. A `logger.warning()` that fires ONLY when any individual healthy result has `latency_ms > 5000` — indicating a timeout-duration value may have been miscategorized as healthy

**Constraint:** At `logging.INFO` level during normal operation (all latencies under 5000 ms), the instrumentation must produce zero output. It becomes visible only at `logging.DEBUG`, or when the anomaly triggers the warning.

---

# Level 2.5 Solutions

## Solution — Extension A

```python
async def check_service(name: str, url: str) -> HealthResult:
    start = time.perf_counter()
    await asyncio.sleep(0.05)
    elapsed_ms = (time.perf_counter() - start) * 1000

    if name.startswith("down-"):
        result = HealthResult(name, 503, elapsed_ms, error="Service unavailable")
        logger.warning(
            "Health check FAILED: service=%s error=%s latency_ms=%.1f",
            name, result.error, elapsed_ms,
        )
        return result

    result = HealthResult(name, 200, elapsed_ms)
    logger.info(
        "Health check OK: service=%s latency_ms=%.1f",
        name, elapsed_ms,
    )
    return result
```

**Expected log output for `{"api": ..., "down-db": ...}` (order may vary — async):**
```
INFO  Health check OK: service=api latency_ms=50.3
WARNING  Health check FAILED: service=down-db error=Service unavailable latency_ms=50.1
```

Why not `print()`: In an async server process, stdout may be interleaved incoherently across concurrent requests and is not captured in log files. `logger.warning()` is visible by default at production log levels; `logger.info()` can be silenced in production by setting `level=logging.WARNING` — without touching a single line of source code.

---

## Solution — Extension B

```python
def build_summary(results: list[HealthResult]) -> dict:
    assert len(results) > 0, (
        "build_summary() received an empty results list. "
        "Call run_health_checks() first and ensure it returns at least one result."
    )
    healthy = [r for r in results if r.is_healthy]
    avg = round(
        sum(r.latency_ms for r in healthy) / len(healthy), 2
    ) if healthy else 0.0

    summary = {
        "total":          len(results),
        "healthy":        len(healthy),
        "degraded":       len(results) - len(healthy),
        "avg_latency_ms": avg,
    }

    assert summary["avg_latency_ms"] >= 0.0, (
        f"avg_latency_ms is {summary['avg_latency_ms']!r} — a negative latency indicates "
        "a bug in the perf_counter measurement."
    )
    assert summary["healthy"] + summary["degraded"] == summary["total"], (
        f"Partitioning invariant violated: "
        f"healthy={summary['healthy']} + degraded={summary['degraded']} "
        f"!= total={summary['total']}"
    )

    return summary
```

`test_empty_healthy_returns_zero_latency` passes a list containing ONE unhealthy result — not an empty list — so the first assertion does not fire. ✓

---

## Solution — Extension C

```python
def build_summary(results: list[HealthResult]) -> dict:
    assert len(results) > 0, "build_summary() received an empty results list."

    healthy = [r for r in results if r.is_healthy]

    if healthy:
        latency_values = [r.latency_ms for r in healthy]
        latency_sum    = sum(latency_values)

        # Visible only at DEBUG level — zero output during normal INFO operation
        logger.debug(
            "build_summary latency breakdown: count=%d values=%s sum=%.4f",
            len(healthy), latency_values, latency_sum,
        )

        # Fires at WARNING even at INFO level — anomaly condition
        anomalous = [r for r in healthy if r.latency_ms > 5000]
        if anomalous:
            logger.warning(
                "ANOMALY: %d healthy service(s) have latency > 5000 ms — "
                "possible timeout result miscategorized as healthy: %s",
                len(anomalous),
                [(r.service_name, round(r.latency_ms, 1)) for r in anomalous],
            )

    avg = round(sum(r.latency_ms for r in healthy) / len(healthy), 2) if healthy else 0.0

    summary = {
        "total":          len(results),
        "healthy":        len(healthy),
        "degraded":       len(results) - len(healthy),
        "avg_latency_ms": avg,
    }

    assert summary["avg_latency_ms"] >= 0.0, "avg_latency_ms is negative — measurement bug."
    assert summary["healthy"] + summary["degraded"] == summary["total"], "Partitioning invariant violated."

    return summary
```

At `INFO` level during normal operation: the `logger.debug()` call produces no output. If a healthy service ever reports `latency_ms > 5000`, the `logger.warning()` fires immediately, capturing the service name and measured value — without a single breakpoint or code change needed after the fact.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## Scenario

You receive an urgent message: *"Our batch processor silently dropped 17 of 200 tasks last night. We don't know which ones failed, why they failed, or what exception was raised."*

You SSH into the server. Your access is:
- ✅ Read the production source code
- ✅ Run a separate diagnostic script in the same Python environment
- ✅ Use `logging`, `asyncio`, and Python's standard library
- ❌ No GUI or IDE debugger
- ❌ Cannot modify the production code
- ❌ Cannot attach `breakpoint()` to any running process

**The production code (read-only):**

```python
# batch_processor.py  ← DO NOT MODIFY

import asyncio
from dataclasses import dataclass

@dataclass
class Task:
    task_id: str
    data: dict
    priority: int = 0

async def process_task(task: Task) -> dict:
    """Process one task. May raise on failure conditions."""
    await asyncio.sleep(0.01)

    if task.priority < 0:
        raise ValueError(f"Invalid priority for task {task.task_id}: {task.priority}")

    if "payload" not in task.data:
        raise KeyError("payload")

    return {"task_id": task.task_id, "result": len(task.data["payload"]), "status": "ok"}

async def run_batch(tasks: list[Task]) -> list[dict]:
    """Runs all tasks concurrently. Silently drops failures per design."""
    results = []
    for coro in asyncio.as_completed([process_task(t) for t in tasks]):
        try:
            result = await coro
            results.append(result)
        except Exception:
            pass    # ← cannot change this
    return results
```

---

**Task 3a: Write a diagnostic script**

Write `diagnose.py` — a standalone script that imports from `batch_processor` and:
1. Creates a test batch where you know IN ADVANCE which tasks will fail, using the failure conditions described in the code comments
2. Calls `run_batch()` with this test batch
3. Compares expected successful IDs to returned IDs to determine which were dropped
4. Reports the drop rate using `logging`
5. Uses only `logging`, `asyncio`, and the imported module

You may NOT use any interactive tool.

---

**Task 3b: Observability wrapper**

Write `run_batch_observed` — a wrapper that calls `run_batch(tasks)` without modifying it, and:
- Before the call: logs the batch size at INFO level
- After the call: logs how many results were received vs expected
- If results < submitted: emits a `logger.warning()` with the exact discrepancy count

```python
async def run_batch_observed(tasks: list[Task]) -> list[dict]:
    ...
```

---

**Task 3c: Identify which tasks were dropped**

The team now needs the specific `task_id` values of dropped tasks, so they can be re-queued. Implement `run_batch_with_audit` — you cannot modify `run_batch`; you must infer dropped IDs externally by comparing input to output.

```python
async def run_batch_with_audit(
    tasks: list[Task],
) -> tuple[list[dict], list[str]]:
    """
    Returns: (results, dropped_task_ids)
    Cannot modify run_batch. Must infer dropped IDs from outside.
    """
    ...
```

Add one assertion that verifies the audit invariant: every submitted task is accounted for in either results or dropped_task_ids.

---

# Level 3 Solutions

## Solution 3a: Diagnostic Script

```python
# diagnose.py
import asyncio
import logging
from batch_processor import Task, run_batch

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)
logger = logging.getLogger(__name__)

async def main() -> None:
    # Build a controlled batch where we know which tasks will fail:
    tasks = [
        Task("ok-001",   {"payload": "alpha"},   priority=1),   # succeeds
        Task("ok-002",   {"payload": "beta"},    priority=0),   # succeeds
        Task("fail-001", {"payload": "gamma"},   priority=-1),  # fails: priority < 0
        Task("fail-002", {"no_payload": "data"}, priority=1),   # fails: missing 'payload'
        Task("ok-003",   {"payload": "delta"},   priority=2),   # succeeds
    ]

    # Determine which IDs should succeed based on known failure conditions
    expected_ids = {
        t.task_id for t in tasks
        if t.priority >= 0 and "payload" in t.data
    }

    logger.info("Submitting %d tasks. Expecting %d successes.", len(tasks), len(expected_ids))

    results = await run_batch(tasks)

    returned_ids = {r["task_id"] for r in results}
    dropped_ids  = expected_ids - returned_ids

    logger.info("Results received: %d", len(results))
    if dropped_ids:
        logger.warning(
            "DROPPED: %d tasks that should have succeeded were not returned: %s",
            len(dropped_ids), sorted(dropped_ids),
        )

    total_dropped = len(tasks) - len(results)
    drop_rate     = total_dropped / len(tasks) * 100
    logger.info("Overall drop rate: %.1f%% (%d/%d tasks)", drop_rate, total_dropped, len(tasks))

asyncio.run(main())
```

**Expected output:**
```
INFO  Submitting 5 tasks. Expecting 3 successes.
INFO  Results received: 3
INFO  Overall drop rate: 40.0% (2/5 tasks)
```

---

## Solution 3b: Observability Wrapper

```python
import logging
from batch_processor import Task, run_batch

logger = logging.getLogger(__name__)

async def run_batch_observed(tasks: list[Task]) -> list[dict]:
    submitted = len(tasks)
    logger.info("Batch starting: %d tasks submitted.", submitted)

    results = await run_batch(tasks)

    returned = len(results)
    dropped  = submitted - returned

    if dropped > 0:
        logger.warning(
            "Batch complete: %d/%d tasks were dropped (%.1f%% drop rate).",
            dropped, submitted, dropped / submitted * 100,
        )
    else:
        logger.info("Batch complete: all %d tasks returned results.", submitted)

    return results
```

---

## Solution 3c: Audit Wrapper

```python
async def run_batch_with_audit(
    tasks: list[Task],
) -> tuple[list[dict], list[str]]:
    submitted_ids = {t.task_id for t in tasks}

    results = await run_batch(tasks)

    returned_ids = {r["task_id"] for r in results}
    dropped_ids  = list(submitted_ids - returned_ids)

    assert len(results) + len(dropped_ids) == len(tasks), (
        f"Audit invariant violated: "
        f"{len(results)} results + {len(dropped_ids)} dropped "
        f"!= {len(tasks)} submitted. "
        "A task_id may appear in results that was not in the submitted list."
    )

    return results, dropped_ids
```

**Why this works:** Every submitted task has a `task_id`. Every result dict also has a `task_id`. The set difference `submitted_ids - returned_ids` gives exactly the IDs that were submitted but whose result never appeared — because `run_batch` only appends successful results and never fabricates task IDs. The audit happens entirely outside `run_batch`, using only input/output comparison.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## Prompt

*"The IDE debugger is always the best debugging tool. It shows all variables, pauses execution, and doesn't require code changes. For any bug a developer will encounter, the debugger gives more information than logging or any other tool."*

Describe TWO specific scenarios where attaching the IDE debugger makes debugging **harder** — not just inconvenient, but where the debugger actively conceals the bug or makes it impossible to observe. For each scenario: describe the situation, provide a minimal code snippet, explain exactly why the debugger conceals the bug, and name the correct tool.

---

**Scenario 1: The Race Condition That Vanishes**

```python
import threading

shared_counter = 0

def handle_request() -> None:
    global shared_counter
    current   = shared_counter          # Step 1: READ
    # ← if two threads are BOTH here simultaneously, both read the same value
    new_value = current + 1             # Step 2: COMPUTE
    shared_counter = new_value          # Step 3: WRITE

threads = [threading.Thread(target=handle_request) for _ in range(20)]
for t in threads: t.start()
for t in threads: t.join()

print(shared_counter)   # Expected: 20. Actual (without debugger): often 14–19.
```

**Why the debugger conceals it:** When you set a breakpoint inside `handle_request`, the debugger pauses execution and serializes what should be concurrent execution — only the thread that hit the breakpoint runs while you inspect variables. The race condition exists only when two threads occupy the READ-to-WRITE window simultaneously. The debugger eliminates that window by pausing. The bug vanishes under observation. This is the heisenbug pattern: adding observation changes behavior.

**Correct tool:** Logging with high-resolution timestamps. Add `logger.debug("[%.6f] thread=%s READ=%d WRITE=%d", time.perf_counter(), threading.current_thread().name, current, new_value)`. Run at full speed under concurrent load. Read the log afterward — two threads logging the same READ value before either WRITE is the race, captured in the record.

---

**Scenario 2: The Performance Measurement That Lies**

```python
import cProfile, pstats

def process_large_dataset(items: list[str]) -> list[str]:
    """This has a hidden O(n²) operation inside."""
    seen = []
    result = []
    for item in items:
        if item not in seen:   # ← O(n) lookup on a list — O(n²) total
            seen.append(item)
            result.append(item)
    return result

# Profiling to find the bottleneck in a 50,000-item dataset
with cProfile.Profile() as pr:
    process_large_dataset([f"item_{i % 1000}" for i in range(50_000)])

pstats.Stats(pr).sort_stats("cumulative").print_stats(5)
```

**Why the debugger conceals it:** Every Python bytecode operation incurs additional overhead when executed under the debugger. A function that takes 40 ms normally might appear to take 400 ms under the debugger. If you set a breakpoint inside the loop and step through, each `Step Over` takes tens of milliseconds of debugger overhead — the 50,000-iteration loop would take hours. Any timing comparison made inside a debugger session is contaminated and meaningless.

**Correct tool:** `cProfile` run OUTSIDE the debugger on the actual data scale, as shown above. `pstats` sorted by `cumulative` will show the `__contains__` method on `list` consuming the majority of time — the O(n) lookup inside the loop. The debugger is the wrong instrument entirely for any timing-sensitive measurement.

---

# LEVEL 5: THE TEACH BACK

## Scenario

A junior developer shows you this function and says: *"This is production-ready. It can never crash, I can always re-debug it when needed, and I've added output so I can see what's happening. What could possibly go wrong?"*

```python
# payment_processor.py
from dataclasses import dataclass

@dataclass
class Transaction:
    tx_id: str
    amount: float
    currency: str

def fetch_exchange_rate(currency: str) -> float:
    rates = {"USD": 1.0, "EUR": 1.08, "GBP": 1.27}
    if currency not in rates:
        raise KeyError(f"Unsupported currency: {currency}")
    return rates[currency]

def process_transaction(tx: Transaction) -> dict:
    try:
        print(f"Processing {tx.tx_id}...")   # ← "output so I can see"
        breakpoint()                          # ← "can always re-debug"

        rate      = fetch_exchange_rate(tx.currency)
        converted = tx.amount * rate

        return {
            "tx_id":             tx.tx_id,
            "converted_amount":  converted,
            "rate_applied":      rate,
        }

    except Exception:                         # ← "can never crash"
        return {"tx_id": tx.tx_id, "status": "failed", "reason": "processing error"}
```

Write a professional, technically precise explanation that addresses all four problems. For each: identify the problem, explain the specific harm it causes in production (not "it's bad practice," but what will actually break and when), then provide the corrected version.

---

# Level 5 Solution

## The Four Problems

**Problem 1: `print(f"Processing {tx.tx_id}...")` — print instead of logging**

In a production FastAPI server or Celery worker, the process's stdout may be:
- Discarded entirely if the process is daemonized or run under a process manager (`systemd`, `supervisord`)
- Interleaved incoherently with output from 50 concurrent requests, making individual transaction traces unreadable
- Lost between process restarts with no persistence
- Impossible to filter — you cannot silence debug prints in production without a code change and a redeployment

When the 2 AM production incident happens and you need to know what happened to transaction `tx-88234`, the `print()` output is either gone or buried in 10,000 interleaved lines with no way to search for just that transaction. `logger.info("Processing: tx_id=%s", tx.tx_id)` survives in the log file, is searchable by tx_id, and can be silenced in production by setting `level=logging.WARNING` without touching a line of source code.

**Problem 2: `breakpoint()` — left in production code**

`breakpoint()` is not a comment or a conditional flag. It is executable code that calls `sys.breakpointhook()`, which by default launches `pdb`. In a running server process with no terminal attached, `pdb` opens and waits for keyboard input that will never arrive. The ENTIRE REQUEST hangs at that line indefinitely. With concurrent load, every request that reaches `process_transaction` hangs. The process accumulates open connections, exhausts its thread pool or connection pool, and eventually becomes completely unresponsive — all while appearing to be "running" from the process manager's perspective.

This is not a hypothetical risk. Leaving `breakpoint()` in a deployed codebase is equivalent to inserting `while True: pass` on that line.

**Problem 3 & 4: `except Exception: return {"reason": "processing error"}` — swallowing all exceptions**

This has two distinct harms:

*Harm A — Bugs become invisible.* If `fetch_exchange_rate` is called with an unsupported currency like `"JPY"`, it raises `KeyError: "Unsupported currency: JPY"`. The `except Exception` catches this and returns `{"status": "failed", "reason": "processing error"}`. The calling code sees "failed" but has no idea whether the failure was due to an unsupported currency, a network outage, or a typo in the code. The specific, actionable error is destroyed.

*Harm B — The return shape is inconsistent.* The success path returns `{"tx_id", "converted_amount", "rate_applied"}`. The failure path returns `{"tx_id", "status", "reason"}`. Any caller that does `result["converted_amount"]` will get a `KeyError` on the failure case. The exception was "caught" inside `process_transaction`, but propagated outward in a different form — now the traceback points to the CALLER, not to the actual failure site. Debugging becomes harder, not easier.

---

## The Corrected Version

```python
import logging
from dataclasses import dataclass

logger = logging.getLogger(__name__)

@dataclass
class Transaction:
    tx_id: str
    amount: float
    currency: str

class UnsupportedCurrencyError(ValueError):
    """Raised when a transaction uses a currency the system does not support."""

def fetch_exchange_rate(currency: str) -> float:
    rates = {"USD": 1.0, "EUR": 1.08, "GBP": 1.27}
    if currency not in rates:
        raise UnsupportedCurrencyError(
            f"Currency '{currency}' is not supported. "
            f"Accepted currencies: {sorted(rates.keys())}"
        )
    return rates[currency]

def process_transaction(tx: Transaction) -> dict:
    # No breakpoint() — use the IDE debugger locally with a launch.json config,
    # or pdb on a remote server. Never leave breakpoint() in committed code.

    logger.info(
        "Processing transaction: tx_id=%s currency=%s amount=%.2f",
        tx.tx_id, tx.currency, tx.amount,
    )

    try:
        rate = fetch_exchange_rate(tx.currency)   # raises UnsupportedCurrencyError only
    except UnsupportedCurrencyError as e:
        logger.warning("Transaction rejected: tx_id=%s reason=%s", tx.tx_id, e)
        return {
            "tx_id":   tx.tx_id,
            "status":  "failed",
            "reason":  str(e),         # specific, actionable message
        }

    # Any OTHER exception — unexpected error, network failure, bug in this module —
    # is NOT caught. It propagates to the FastAPI/Celery error handler, which logs
    # the full traceback. This is exactly what we want: unexpected failures crash
    # loudly and visibly rather than silently returning wrong data.

    converted = tx.amount * rate
    logger.info(
        "Transaction complete: tx_id=%s converted=%.2f rate=%.4f",
        tx.tx_id, converted, rate,
    )
    return {
        "tx_id":            tx.tx_id,
        "converted_amount": converted,
        "rate_applied":     rate,
        "status":           "ok",
    }
```

**What changed and why:**

| Before | After | Why It Matters |
|---|---|---|
| `print(f"Processing...")` | `logger.info(...)` with tx_id | Persists in log files; filterable; searchable by transaction ID during incidents |
| `breakpoint()` | Removed entirely | Would freeze every request that hit this line in production |
| `except Exception` | `except UnsupportedCurrencyError` | Only the expected domain error is caught; programming bugs propagate as full tracebacks |
| `"reason": "processing error"` | `"reason": str(e)` | Specific, actionable; names the currency that was rejected |
| No logging on failure | `logger.warning(...)` on failure | Operations teams can alert on elevated warning rates; failure is visible without needing to check the database |

The relationship to this lecture's central thesis: `except Exception` creates exactly the gap between "what you think your code does" and "what it actually does" that the entire lecture is about closing. The swallowed exception removes the traceback — the only piece of evidence that would let you close that gap. Every technique from today's lecture (traceback reading, the debugger, logging, hypothesis-driven diagnosis) exists to make reality visible. Catching `Exception` makes it invisible.