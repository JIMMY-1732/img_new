# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "The ORM is a transparent wrapper. `session.add()` saves to the database, my Python objects always have their data, `default` and `server_default` are interchangeable synonyms, and catching a database error with `try/except` is the same as handling it."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | `session.add()` persists data to the database | `add()` only marks the object as "pending" in the Session's internal tracking. No SQL is sent. |
| 2 | `flush()` and `commit()` are the same thing | `flush()` sends SQL within an **open** transaction. `commit()` finalizes it. Flushed-but-uncommitted data vanishes on rollback or session close. |
| 3 | After `commit()`, my Python object still has all its data | `commit()` **expires** all loaded attributes. Accessing them requires an active session (lazy load) or explicit `refresh()`. |
| 4 | `default` and `server_default` both set values immediately | `default` is Python-side (set at `__init__`). `server_default` is DB-side (set during INSERT). The Python object sees `None` for `server_default` until refreshed. |
| 5 | Catching a DB error means the session is fine | After an unrolled-back database error, the Session's transaction is poisoned. All subsequent operations raise `PendingRollbackError`. |
| 6 | Different queries return different Python objects | The Session's **identity map** guarantees: same primary key = same Python object, regardless of how you queried it. |
| 7 | `create_all()` updates existing tables | It only creates tables that don't exist. Model changes to existing tables are silently ignored. |

---

## The Main Purpose (Essence)

**Core Insight:** The ORM manages a **stateful, transactional workspace** (the Session) that translates between Python objects and SQL operations. This workspace has explicit lifecycle stages — objects go through states (transient → pending → persistent → detached), and the boundary between "in-memory change" and "permanent database change" is drawn by `commit()`, not by `add()` or `flush()`.

**The Problems Being Solved:**

| Component | Without It | With It |
|-----------|-----------|---------|
| Engine | Manually open/close connections per query | Connection pool managed automatically |
| Model classes | Positional row indexing (`row[3]`), repeated mapping | Named attributes (`user.email`), single mapping definition |
| Session | Manual `BEGIN`/`COMMIT`/`ROLLBACK`, manual change tracking | Automatic dirty tracking, transaction context manager |
| `echo=True` | ORM is a black box; you hope the SQL is correct | Every generated statement is visible; no magic |

**The Key Distinction:**
- `session.add()` = "put it in the shopping cart" (no SQL)
- `session.flush()` = "scan items at register" (SQL sent, transaction **open**)
- `session.commit()` = "pay and leave" (transaction **finalized**, changes permanent)
- `session.rollback()` = "abandon the cart" (everything undone)

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "Different queries = different objects" | Modifying `product_a` doesn't affect `product_b` | Both variables point to the same Python object; change is visible through both | Identity map caches objects by primary key across all query types |
| "`flush()` saves data" | Flushed data survives session close | Data vanishes — transaction was never committed | `flush()` sends SQL within an open transaction; `commit()` is what finalizes it |
| "Both `default` and `server_default` set values at creation" | `entry.created_at` has a timestamp immediately after construction | `entry.created_at` is `None` until the DB processes INSERT and you `refresh()` | `default` runs in Python at `__init__`; `server_default` runs in the DB at INSERT time |
| "Catching `IntegrityError` handles the problem" | Session continues working after `except IntegrityError` | Next operation raises `PendingRollbackError` | DB transaction is invalidated after error; Session requires explicit `rollback()` before reuse |
| "Returning an object after commit works" | Caller accesses `user.name` normally | `DetachedInstanceError` — attributes expired, session closed | `commit()` expires attributes; `close()` detaches object; no session for lazy-load |

---

# LEVEL 1: PREDICT & EXPLAIN

## Shared Setup

All Level 1 exercises use this setup. Include it at the top of your script.

```python
from sqlalchemy import create_engine, String, Text, DateTime, func, select
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, Session, sessionmaker,
)
from sqlalchemy.exc import IntegrityError
from datetime import datetime
from typing import Optional

engine = create_engine("sqlite:///:memory:", echo=False)

class Base(DeclarativeBase):
    pass
```

---

## Exercise 1.1: The Invisible Link

```python
class Product(Base):
    __tablename__ = "products"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    price: Mapped[int] = mapped_column(default=0)

    def __repr__(self) -> str:
        return f"Product(id={self.id!r}, name={self.name!r}, price={self.price!r})"

Base.metadata.create_all(engine)

# Seed one product
with Session(engine) as session:
    session.add(Product(name="Widget", price=100))
    session.commit()

# --- The interesting part ---
with Session(engine) as session:
    # Query 1: Load by primary key
    product_a = session.get(Product, 1)

    # Query 2: Load by name (completely different query)
    stmt = select(Product).where(Product.name == "Widget")
    product_b = session.scalars(stmt).first()

    # Modify through reference A only
    product_a.price = 999

    print(f"product_a.price = {product_a.price}")
    print(f"product_b.price = {product_b.price}")
    print(f"Same object? {product_a is product_b}")
```

**Questions:**

1. **Predict the output.** Specifically:
   - What is `product_b.price` after only `product_a.price` was changed?
   - What does `product_a is product_b` return?

2. **Explain why.** A student says: "I used two completely different queries — `session.get()` vs `select().where()`. These are separate database operations, so they must return separate Python objects." Why is this student wrong?

3. **Mental Trace:** When `session.get(Product, 1)` returns a Product, where does the Session store it? When the second query (`select ... where name = 'Widget'`) returns a row with `id=1`, what does the Session do before constructing a new Python object?

---

## Exercise 1.2: The Phantom Row

```python
class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    item: Mapped[str] = mapped_column(String(200))

    def __repr__(self) -> str:
        return f"Order(id={self.id!r}, item={self.item!r})"

Base.metadata.create_all(engine)

# --- Attempt to create an order ---
saved_id = None

try:
    with Session(engine) as session:
        order = Order(item="Laptop")
        session.add(order)
        session.flush()

        saved_id = order.id
        print(f"After flush — order.id = {saved_id}")
        print(f"After flush — order in DB? Let's check...")

        # Verify: query within the same session
        found_inside = session.get(Order, saved_id)
        print(f"Found inside same session: {found_inside}")

        # Simulate a failure before commit
        raise ValueError("Payment processing failed!")
        session.commit()  # Never reached
except ValueError:
    print("Payment error caught. Continuing...\n")

# --- Try to find the order in a new session ---
print(f"Looking for order with id={saved_id}")
with Session(engine) as session:
    found_outside = session.get(Order, saved_id)
    print(f"Found in new session: {found_outside}")
```

**Questions:**

1. **Predict the output.** Specifically:
   - What does `found_inside` show? (Query within the same session after flush.)
   - What does `found_outside` show? (Query in a new session after the first session closed.)

2. **Explain why.** The student got an auto-generated `id` back from the database. The row was visible inside the same session. How can a row that *exists in the database* (enough to have an ID) simply vanish?

3. **Mental Trace:** Walk through the transaction lifecycle:
   - What SQL does `session.flush()` send?
   - What is the state of the database transaction at that point?
   - What happens when the `with Session(engine)` block exits due to the `ValueError`?
   - Could a *separate* database connection (e.g., another application) see this row between `flush()` and the session's exit?

---

## Exercise 1.3: The Two Timelines

```python
class LogEntry(Base):
    __tablename__ = "log_entries"
    id: Mapped[int] = mapped_column(primary_key=True)
    message: Mapped[str] = mapped_column(String(500))
    level: Mapped[str] = mapped_column(String(20), default="INFO")
    created_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime, server_default=func.now()
    )

    def __repr__(self) -> str:
        return f"LogEntry(id={self.id!r}, level={self.level!r})"

Base.metadata.create_all(engine)

# --- Trace the values at each stage ---
entry = LogEntry(message="System started")

print("Step 1 — After Python object creation (no session yet):")
print(f"  id:         {entry.id!r}")
print(f"  level:      {entry.level!r}")
print(f"  created_at: {entry.created_at!r}")

with Session(engine) as session:
    session.add(entry)

    print("\nStep 2 — After session.add():")
    print(f"  id:         {entry.id!r}")
    print(f"  level:      {entry.level!r}")
    print(f"  created_at: {entry.created_at!r}")

    session.commit()
    session.refresh(entry)

    print("\nStep 3 — After session.commit() + session.refresh():")
    print(f"  id:         {entry.id!r}")
    print(f"  level:      {entry.level!r}")
    print(f"  created_at: {entry.created_at!r}")
```

**Questions:**

1. **Predict the output for all three steps.** For each of `id`, `level`, and `created_at`, state the exact value (or `None`) and whether it changes between steps.

2. **Explain the discrepancy.** Both `level` and `created_at` have "defaults" defined on the model. A student expects both to have values at Step 1. Why does `level` have a value but `created_at` does not?

3. **Mental Trace:** For each field:
   - At what *exact moment* is the value determined — when the Python constructor runs, when `flush()` sends the INSERT, or when you `refresh()` from the database?
   - Which layer (Python or Database) is responsible for supplying the value?
   - Why is `session.add()` (Step 2) a complete no-op for all three values?

---

## Exercise 1.4: The Poison Pill

```python
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(50), unique=True)
    email: Mapped[str] = mapped_column(String(255), unique=True)

    def __repr__(self) -> str:
        return f"User(id={self.id!r}, username={self.username!r})"

Base.metadata.create_all(engine)

with Session(engine) as session:
    # Step 1: Create Alice — succeeds
    session.add(User(username="alice", email="alice@example.com"))
    session.commit()
    print("Step 1: Alice created.")

    # Step 2: Try to create a duplicate Alice — catch the error
    try:
        session.add(User(username="alice", email="different@example.com"))
        session.commit()
    except IntegrityError:
        print("Step 2: Duplicate caught. No worries, moving on...")

    # Step 3: Create Bob — should be fine, right?
    try:
        session.add(User(username="bob", email="bob@example.com"))
        session.commit()
        print("Step 3: Bob created.")
    except Exception as e:
        print(f"Step 3: FAILED — {type(e).__name__}")
```

**Questions:**

1. **Predict the output.** Does Bob get created? If not, what exception occurs at Step 3 and why?

2. **Explain why.** The student argues: "I caught the `IntegrityError` with `try/except`. That means the error was handled. Python's exception mechanism guarantees the program continues normally after the `except` block." Why is this reasoning wrong in the context of the ORM Session?

3. **Mental Trace:** After the `IntegrityError` is raised inside `session.commit()`:
   - What state is the database transaction in?
   - What state is the Session object in?
   - What must happen before the Session can execute any further SQL?
   - How does this differ from catching a regular Python exception like `ValueError`?

---

# Level 1 Solutions

## Solution 1.1: The Invisible Link

**1. Output:**
```
product_a.price = 999
product_b.price = 999
Same object? True
```

**2. Explain why:**

The Session maintains an **identity map** — an internal dictionary keyed by `(table_name, primary_key)`. When `session.get(Product, 1)` runs, SQLAlchemy loads the row, constructs a `Product` object, and stores it in the identity map under key `("products", 1)`.

When the second query (`select ... where name = 'Widget'`) executes, the database returns a row with `id=1`. Before constructing a *new* Python object, the Session checks its identity map: "Do I already have a Product with `id=1`?" It does. So it returns **the same Python object** it created earlier — it does **not** create a second instance.

This means `product_a` and `product_b` are not just "equal" — they are literally the same object in memory. Modifying `product_a.price` and reading `product_b.price` are the same memory access.

**3. Mental Trace:**
```
1. session.get(Product, 1)
   → SQL: SELECT ... FROM products WHERE id = 1
   → Constructs Product(id=1, name='Widget', price=100)
   → Stores in identity map: {("products", 1): <Product object at 0xABC>}
   → Returns <Product object at 0xABC>

2. select(Product).where(Product.name == "Widget")
   → SQL: SELECT ... FROM products WHERE name = 'Widget'
   → DB returns row with id=1
   → Checks identity map: ("products", 1) → already exists!
   → Returns <Product object at 0xABC>  (SAME object, no new construction)

3. product_a.price = 999
   → Modifies the ONLY Product(id=1) object in memory
   → product_b references this same object
   → product_b.price is also 999
```

**The Key Insight:**
The identity map exists to **prevent inconsistencies within a session**. If two queries returned separate objects for the same row, you could modify one and not the other, leading to conflicting changes. The identity map guarantees: one row = one object = one source of truth per session.

---

## Solution 1.2: The Phantom Row

**1. Output:**
```
After flush — order.id = 1
After flush — order in DB? Let's check...
Found inside same session: Order(id=1, item='Laptop')
Payment error caught. Continuing...

Looking for order with id=1
Found in new session: None
```

**2. Explain why:**

The row was real — it had an auto-generated ID, and it was visible inside the same session. But it existed **inside an open transaction that was never committed.**

`session.flush()` sends the `INSERT INTO orders ...` SQL to the database. The database processes it, assigns `id=1`, and sends it back. But this all happens within a **BEGIN...** block (an open transaction). The database has provisionally written the row, but it hasn't executed **COMMIT**. The row is visible to the current transaction (the current session's connection), but **no other connection can see it**.

When the `ValueError` causes the `with Session(...)` block to exit via an exception, the Session's `__exit__` method triggers a **ROLLBACK**. The database undoes the INSERT. The row never existed from the perspective of any future connection.

**3. Mental Trace:**
```
1. session.flush()
   → SQL: BEGIN (implicit)
   → SQL: INSERT INTO orders (item) VALUES ('Laptop') RETURNING id
   → DB assigns id=1, row is provisionally written
   → Transaction is OPEN (not committed)

2. session.get(Order, 1) inside SAME session
   → Uses the SAME database connection, SAME transaction
   → Can see its own uncommitted data → finds the row

3. raise ValueError(...)
   → Exception propagates to `with Session(...)` __exit__
   → Session detects exception → ROLLBACK
   → SQL: ROLLBACK
   → Database undoes the INSERT. Row is gone.

4. New session → new connection → new transaction
   → SELECT ... WHERE id = 1 → no row found → None
```

**Could another connection see the row between flush and exit?**
No. Transaction isolation (the "I" in ACID, which students learned in Week 5) ensures that uncommitted rows are invisible to other connections (under default READ COMMITTED isolation).

**The Key Insight:**
`flush()` proves that SQL was *generated and sent*. But `commit()` is what makes changes *permanent*. An ID assigned by the database during a flushed INSERT is real **within the transaction**, but it becomes permanently allocated only after commit.

---

## Solution 1.3: The Two Timelines

**1. Output:**
```
Step 1 — After Python object creation (no session yet):
  id:         None
  level:      'INFO'
  created_at: None

Step 2 — After session.add():
  id:         None
  level:      'INFO'
  created_at: None

Step 3 — After session.commit() + session.refresh():
  id:         1
  level:      'INFO'
  created_at: datetime.datetime(2025, 3, 15, 14, 30, 0)  # (actual current time)
```

**2. Explain the discrepancy:**

`level` and `created_at` both have "defaults," but they are implemented in **different layers**:

- **`default="INFO"`** is a **Python-side default**. When SQLAlchemy generates the `__init__` method for `LogEntry`, it treats `default="INFO"` like a dataclass default. The moment you write `LogEntry(message="System started")`, Python's constructor sets `self.level = "INFO"`. No database involved. This is why `level` has a value at Step 1.

- **`server_default=func.now()`** is a **database-side default**. It tells SQLAlchemy: "When generating the CREATE TABLE SQL, add `DEFAULT CURRENT_TIMESTAMP` to this column." Python doesn't set this value. The database does — but only when it processes an INSERT statement. Since no INSERT has happened at Step 1, `created_at` is `None`.

**3. Mental Trace per field:**

| Field | Value at Step 1 | Set by | When |
|---|---|---|---|
| `id` | `None` | Database (auto-increment) | During INSERT (after flush) |
| `level` | `'INFO'` | Python (`__init__`) | At object construction time |
| `created_at` | `None` | Database (`DEFAULT NOW()`) | During INSERT (after flush) |

**Why is `session.add()` (Step 2) a no-op for all values?**
`session.add()` only registers the object in the Session's internal "new objects" list. It does not send SQL, does not assign IDs, does not trigger any defaults. It is purely a bookkeeping operation. SQL only runs at `flush()` or `commit()`.

**The Key Insight:**
Think of `default` as a Python `__init__` parameter. Think of `server_default` as a SQL `DEFAULT` clause. They operate in different runtimes (Python interpreter vs. database server) at different times (object construction vs. INSERT processing).

---

## Solution 1.4: The Poison Pill

**1. Output:**
```
Step 1: Alice created.
Step 2: Duplicate caught. No worries, moving on...
Step 3: FAILED — PendingRollbackError
```

Bob is **not** created.

**2. Explain why:**

Python's `try/except` catches the **Python exception object** — it prevents the program from crashing. But it does nothing about the **database transaction state**.

When `session.commit()` fails due to an `IntegrityError`:
1. The database rejects the INSERT (UNIQUE constraint violation)
2. The database transaction enters an **error state**
3. SQLAlchemy's Session records that its transaction needs a rollback

After `except IntegrityError:` runs, the Python exception is handled, but the Session's internal transaction is still in the error state. The Session refuses to execute any further SQL until you explicitly call `session.rollback()` to clear the failed transaction and begin a fresh one.

This is fundamentally different from catching a regular Python exception like `ValueError`, which has no effect on any external system state.

**3. Mental Trace:**
```
Step 2 internals:
  1. session.add(User(username="alice", ...)) → adds to pending list
  2. session.commit() → triggers flush → sends INSERT
  3. Database: UNIQUE violation → rejects INSERT → returns error
  4. SQLAlchemy: receives error → marks session as "needs rollback"
  5. IntegrityError raised in Python
  6. except IntegrityError: catches the Python exception
  7. Session is still marked as "needs rollback"

Step 3:
  1. session.add(User(username="bob", ...)) → may add to pending list
  2. session.commit() → Session checks its state
  3. State = "needs rollback" → refuses to proceed
  4. Raises PendingRollbackError
```

**The fix:**
```python
    except IntegrityError:
        session.rollback()  # ← MUST reset the transaction
        print("Step 2: Duplicate caught and rolled back.")
```

**The Key Insight:**
In ORM error handling, catching the exception is **not enough**. You must also **restore the Session's transaction state** via `session.rollback()`. Think of it like a car crash: calling a tow truck (except) handles the immediate emergency, but you still have to clear the wreckage from the road (rollback) before traffic (further operations) can resume.

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Haunted Return

**Context:** A developer wrote a helper function to create users. The function works — data appears in the database — but callers keep getting mysterious errors.

```python
from sqlalchemy import create_engine, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session, sessionmaker

engine = create_engine("sqlite:///:memory:", echo=False)

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(50), unique=True)
    email: Mapped[str] = mapped_column(String(255))

    def __repr__(self) -> str:
        return f"User(id={self.id!r}, username={self.username!r})"

Base.metadata.create_all(engine)
SessionLocal = sessionmaker(bind=engine)


def create_user(username: str, email: str) -> User:
    """Create a new user and return it."""
    with SessionLocal() as session:
        user = User(username=username, email=email)
        session.add(user)
        session.commit()
        return user


# Usage
user = create_user("alice", "alice@example.com")

# Verify user is in the database
with SessionLocal() as session:
    found = session.get(User, 1)
    print(f"In database: {found}")  # Works! User exists.

# Try to use the returned object
try:
    print(f"Returned user: {user.username}")
except Exception as e:
    print(f"ERROR: {type(e).__name__}: {e}")
```

---

**Task 2a: Identify the Flaw**

The user IS created in the database (the verification query proves it). But the returned `user` object is unusable. Run the code and observe what happens.

1. What exception is raised when accessing `user.username`?
2. The data exists in the database. Why can't the Python object access its own attributes?

Do NOT fix the code yet.

---

**Task 2b: Explain the Mechanism**

Walk through what happens to the `user` object at each step:

1. After `session.add(user)` — what state is the object in?
2. After `session.commit()` — what happens to the object's attributes?
3. After `return user` and the `with` block exits — what state is the object now?
4. When the caller accesses `user.username` — what does SQLAlchemy's attribute descriptor try to do? Why does it fail?

---

**Task 2c: Instrument to Prove Diagnosis**

Add instrumentation inside `create_user` to trace the object's state at each stage. Use the `sqlalchemy.inspect` function:

```python
from sqlalchemy import inspect

def create_user(username: str, email: str) -> User:
    with SessionLocal() as session:
        user = User(username=username, email=email)

        # Instrument: check state at each stage
        insp = inspect(user)
        print(f"After construction: {insp.transient=}, {insp.pending=}, {insp.persistent=}")
        print(f"  username in __dict__: {'username' in user.__dict__}")

        session.add(user)
        insp = inspect(user)
        print(f"After add: {insp.transient=}, {insp.pending=}, {insp.persistent=}")

        session.commit()
        insp = inspect(user)
        print(f"After commit: {insp.persistent=}, {insp.expired=}")
        print(f"  username in __dict__: {'username' in user.__dict__}")

        return user
```

Run this. What does `insp.expired` tell you? What does the `__dict__` check reveal about where the attribute data went?

---

**Task 2d: Fix the Code**

Fix `create_user` so that the returned `User` object is usable by the caller. Provide **two** different valid approaches.

Hint for approach 1: What Session method reloads an object's attributes from the database?
Hint for approach 2: What if the returned value wasn't the ORM object at all?

---

**Task 2e: Write Verification Test**

Write a test that **fails** on the original (broken) `create_user` and **passes** on your fixed version:

```python
def test_returned_user_is_usable():
    """Returned user object must have accessible attributes."""
    user = create_user("testuser", "test@example.com")

    # These must not raise any exceptions
    assert user.username == "testuser"
    assert user.email == "test@example.com"
    assert user.id is not None
```

---

## Exercise 2.2: The Vanishing Balance

**Context:** A developer wrote a bank transfer function. The code runs without exceptions during normal transfers, but money vanishes when certain edge cases occur.

```python
from sqlalchemy import create_engine, String, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session, sessionmaker
from typing import Optional

engine = create_engine("sqlite:///:memory:", echo=False)

class Base(DeclarativeBase):
    pass

class Account(Base):
    __tablename__ = "accounts"
    id: Mapped[int] = mapped_column(primary_key=True)
    owner: Mapped[str] = mapped_column(String(100))
    balance: Mapped[int] = mapped_column(default=0)

    def __repr__(self) -> str:
        return f"Account(id={self.id!r}, owner={self.owner!r}, balance={self.balance!r})"

Base.metadata.create_all(engine)
SessionLocal = sessionmaker(bind=engine)


def get_balance(session: Session, account_id: int) -> int:
    account = session.get(Account, account_id)
    return account.balance if account else 0


def transfer(session: Session, from_id: int, to_id: int, amount: int) -> None:
    """Transfer amount from one account to another."""
    from_account = session.get(Account, from_id)
    to_account = session.get(Account, to_id)

    if from_account is None:
        raise ValueError(f"Source account {from_id} not found")

    if from_account.balance < amount:
        raise ValueError("Insufficient funds")

    # Debit the source
    from_account.balance -= amount
    session.commit()

    # Credit the destination
    if to_account is None:
        raise ValueError(f"Destination account {to_id} not found")

    to_account.balance += amount
    session.commit()


# --- Test scenario ---
with SessionLocal() as session:
    session.add(Account(owner="Alice", balance=1000))
    session.add(Account(owner="Bob", balance=500))
    session.commit()

# Scenario A: Normal transfer
with SessionLocal() as session:
    transfer(session, from_id=1, to_id=2, amount=200)
    print(f"After normal transfer:")
    print(f"  Alice: {get_balance(session, 1)}")
    print(f"  Bob:   {get_balance(session, 2)}")

# Scenario B: Transfer to nonexistent account
with SessionLocal() as session:
    try:
        transfer(session, from_id=1, to_id=999, amount=300)
    except ValueError as e:
        print(f"\nTransfer failed: {e}")

    print(f"After failed transfer:")
    print(f"  Alice: {get_balance(session, 1)}")
    print(f"  Bob:   {get_balance(session, 2)}")

# Check total money in the system
with SessionLocal() as session:
    total = get_balance(session, 1) + get_balance(session, 2)
    print(f"\nTotal money in system: {total}")
    print(f"Expected total: 1500")
    print(f"Money conserved: {total == 1500}")
```

---

**Task 2a: Identify the Flaw**

Run the code. Focus on Scenario B.

1. What happens to Alice's balance after the failed transfer to account 999?
2. What happens to the total money in the system?
3. Where did the money go?

---

**Task 2b: Explain the Mechanism**

Trace through `transfer(session, from_id=1, to_id=999, amount=300)`:

1. Which line deducts Alice's money?
2. Which line permanently saves that deduction?
3. At what point does the function discover account 999 doesn't exist?
4. Why can't we "undo" the deduction after the `ValueError`?

Draw the relationship between the two `session.commit()` calls and database transactions.

---

**Task 2c: Instrument to Prove Diagnosis**

Add instrumentation to show the exact moment money is lost:

```python
def transfer_instrumented(session, from_id, to_id, amount):
    from_account = session.get(Account, from_id)
    to_account = session.get(Account, to_id)

    # ... (add prints showing balance at each stage) ...
    # ... (add prints showing "COMMIT 1" and "COMMIT 2" markers) ...
```

Show the output that proves money disappears between the two commits.

---

**Task 2d: Fix the Code**

Fix the `transfer` function so that:
- If ANY step fails, NO money is moved (atomic operation)
- All validation happens BEFORE any data modification
- Only ONE commit occurs for the entire transfer

Hint: The lecture distinguished between `flush()` (which sends SQL but keeps the transaction open) and `commit()` (which finalizes). Do you need `flush()` here, or is a single `commit()` sufficient?

---

**Task 2e: Write Verification Test**

Write a test that **fails** on the original code and **passes** on your fixed version:

```python
def test_failed_transfer_conserves_money():
    """Money must be conserved even when transfer fails."""
    with SessionLocal() as session:
        # Reset balances
        alice = session.get(Account, 1)
        bob = session.get(Account, 2)
        alice.balance = 1000
        bob.balance = 500
        session.commit()

    with SessionLocal() as session:
        try:
            transfer(session, from_id=1, to_id=999, amount=300)
        except ValueError:
            session.rollback()

    with SessionLocal() as session:
        total = get_balance(session, 1) + get_balance(session, 2)
        assert total == 1500, f"Money leaked! Total is {total}, expected 1500"
        assert get_balance(session, 1) == 1000, "Alice's balance should be unchanged"
```

---

# Level 2 Solutions

## Solution 2.1: The Haunted Return

**Task 2a: Identify the Flaw**

```
In database: User(id=1, username='alice')
ERROR: DetachedInstanceError: Instance <User at 0x...> is not bound to a Session;
attribute refresh operation cannot proceed
```

The user exists in the database, but the returned Python object raises `DetachedInstanceError` when you try to read `.username`.

**Task 2b: Explain the Mechanism**

```
1. session.add(user)
   → Object state: PENDING
   → Attributes in __dict__: username='alice', email='alice@example.com'
   → The Session knows about this object

2. session.commit()
   → SQL: INSERT INTO users ... → data saved to DB
   → SQLAlchemy EXPIRES all attributes on the object
   → __dict__ is cleared of 'username', 'email', etc.
   → Object state: PERSISTENT (but expired)
   → Why? To avoid stale data. The DB might have triggers or
     defaults that changed values. Expiring forces a reload.

3. return user → the `with` block exits → session.close()
   → Object state: DETACHED (no longer associated with any Session)
   → Attributes are still expired (never refreshed)

4. Caller accesses user.username
   → SQLAlchemy's attribute descriptor intercepts the access
   → Sees the attribute is expired → tries to lazy-load (issue SELECT)
   → No active Session → cannot execute SQL
   → Raises DetachedInstanceError
```

**Task 2c: Instrumentation Output**

```
After construction: insp.transient=True, insp.pending=False, insp.persistent=False
  username in __dict__: True
After add: insp.transient=False, insp.pending=True, insp.persistent=False
After commit: insp.persistent=True, insp.expired=True
  username in __dict__: False
```

The critical line: after `commit()`, `username in __dict__` is `False`. The attribute data was **evicted** from the Python object's `__dict__`. It's not that the data is wrong — it's that the data isn't there at all. SQLAlchemy expired it to force a reload, but once the session closes, reload is impossible.

**Task 2d: Fix — Approach 1 (Refresh before return)**

```python
def create_user(username: str, email: str) -> User:
    with SessionLocal() as session:
        user = User(username=username, email=email)
        session.add(user)
        session.commit()
        session.refresh(user)  # ← Reloads ALL attributes into __dict__
        return user
    # After session closes: object is detached, but attributes are
    # populated in __dict__ (refresh loaded them). No lazy-load needed.
```

**Task 2d: Fix — Approach 2 (Return a dict or Pydantic model)**

```python
def create_user(username: str, email: str) -> dict:
    with SessionLocal() as session:
        user = User(username=username, email=email)
        session.add(user)
        session.commit()
        session.refresh(user)
        # Convert to a plain dict — no ORM dependency
        return {
            "id": user.id,
            "username": user.username,
            "email": user.email,
        }
```

In a real FastAPI application (Week 6, Lecture 4), you would convert to a Pydantic response model instead of a dict. The principle is the same: don't let ORM objects escape their Session's scope.

**Task 2e: Verification**

```python
def test_returned_user_is_usable():
    user = create_user("testuser", "test@example.com")
    assert user.username == "testuser"    # Fails on broken version (DetachedInstanceError)
    assert user.email == "test@example.com"
    assert user.id is not None
```

**The Key Insight:**
`session.commit()` is a double-edged sword: it saves your data permanently but also **invalidates** all loaded Python attributes. If you need to use the object after commit, you must `refresh()` it while the session is still open. Alternatively, convert to a session-independent format before the session closes.

---

## Solution 2.2: The Vanishing Balance

**Task 2a: Output**

```
After normal transfer:
  Alice: 800
  Bob:   700

Transfer failed: Destination account 999 not found
After failed transfer:
  Alice: 500
  Bob:   700

Total money in system: 1200
Expected total: 1500
Money conserved: False
```

300 units of currency have **vanished from the system**. Alice was debited, but the credit to the destination never happened because account 999 doesn't exist — and the debit was already committed before the check.

**Task 2b: Mechanism**

```
transfer(session, from_id=1, to_id=999, amount=300):

1. from_account = session.get(Account, 1)  → Alice (balance=800)
2. to_account = session.get(Account, 999)  → None
3. from_account.balance < 300?  → 800 < 300? No. Passes.

4. from_account.balance -= 300  → Alice.balance = 500 (in Python memory)
5. session.commit()             → COMMIT #1: Alice.balance=500 is PERMANENT
   ─── POINT OF NO RETURN ───

6. if to_account is None:       → True! Account 999 doesn't exist.
7. raise ValueError(...)        → Exception raised

   Alice has been debited 300. The credit never happens.
   COMMIT #1 cannot be undone — it's already finalized.
```

The two `session.commit()` calls create **two separate transactions**. The first commits the debit; the second would commit the credit. When the second fails, the first is already permanent.

**Task 2c: Instrumentation Output**

```python
def transfer_instrumented(session, from_id, to_id, amount):
    from_account = session.get(Account, from_id)
    to_account = session.get(Account, to_id)

    print(f"  Before debit:  from={from_account.balance}, to={'N/A' if to_account is None else to_account.balance}")

    from_account.balance -= amount
    session.commit()
    print(f"  COMMIT #1 done. from={from_account.balance} — THIS IS PERMANENT")

    if to_account is None:
        print(f"  Destination missing! But from_account already committed!")
        raise ValueError(f"Destination account {to_id} not found")

    to_account.balance += amount
    session.commit()
    print(f"  COMMIT #2 done. to={to_account.balance}")
```

Output:
```
  Before debit:  from=800, to=N/A
  COMMIT #1 done. from=500 — THIS IS PERMANENT
  Destination missing! But from_account already committed!
```

**Task 2d: Fix**

```python
def transfer(session: Session, from_id: int, to_id: int, amount: int) -> None:
    """Transfer amount from one account to another. Atomic."""
    # 1. Load ALL data first
    from_account = session.get(Account, from_id)
    to_account = session.get(Account, to_id)

    # 2. Validate ALL preconditions before modifying anything
    if from_account is None:
        raise ValueError(f"Source account {from_id} not found")
    if to_account is None:
        raise ValueError(f"Destination account {to_id} not found")
    if from_account.balance < amount:
        raise ValueError("Insufficient funds")

    # 3. Modify both (in memory — no SQL sent yet)
    from_account.balance -= amount
    to_account.balance += amount

    # 4. Single commit — both changes in ONE transaction
    session.commit()
    # If commit fails, BOTH changes are rolled back. Money is conserved.
```

The key changes:

1. **All validation before any modification:** The destination account is checked *before* any balance is changed.
2. **Single commit:** Both the debit and credit are in the same transaction. Either both succeed or both are rolled back.
3. **No flush needed:** Since we don't need intermediate database-generated values (like auto-increment IDs), a single `commit()` at the end is sufficient.

**Task 2e: Verification Test**

```python
def test_failed_transfer_conserves_money():
    # Reset
    with SessionLocal() as session:
        alice = session.get(Account, 1)
        bob = session.get(Account, 2)
        alice.balance = 1000
        bob.balance = 500
        session.commit()

    # Attempt bad transfer
    with SessionLocal() as session:
        try:
            transfer(session, from_id=1, to_id=999, amount=300)
        except ValueError:
            session.rollback()

    # Verify conservation
    with SessionLocal() as session:
        total = get_balance(session, 1) + get_balance(session, 2)
        assert total == 1500, f"Money leaked! Total is {total}"
        assert get_balance(session, 1) == 1000, "Alice should be unchanged"
        # Original code: FAILS (total=1200, Alice=500)
        # Fixed code: PASSES (total=1500, Alice=1000)
```

**The Key Insight:**
Every `session.commit()` is a **transaction boundary** — a point of no return. If a logical operation spans multiple data modifications, all modifications must live within a **single** transaction (a single commit). The rule: validate everything, modify everything, commit once.

---

# LEVEL 2.5: EXTEND EXISTING CODE

## Given: Article Management System

The following code is working and correct. A test suite is provided. **Your extensions must not break the existing tests.**

### Code

```python
# article_system.py
from sqlalchemy import create_engine, String, Text, select
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, Session, sessionmaker,
)
from typing import Optional

engine = create_engine("sqlite:///:memory:", echo=False)


class Base(DeclarativeBase):
    pass


class Article(Base):
    __tablename__ = "articles"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    slug: Mapped[str] = mapped_column(String(200), unique=True)
    body: Mapped[Optional[str]] = mapped_column(Text)
    status: Mapped[str] = mapped_column(String(20), default="draft")

    def __repr__(self) -> str:
        return f"Article(id={self.id!r}, title={self.title!r}, status={self.status!r})"


Base.metadata.create_all(engine)
SessionLocal = sessionmaker(bind=engine)


def create_article(
    session: Session, title: str, slug: str, body: Optional[str] = None
) -> Article:
    article = Article(title=title, slug=slug, body=body)
    session.add(article)
    session.commit()
    session.refresh(article)
    return article


def get_article(session: Session, article_id: int) -> Optional[Article]:
    return session.get(Article, article_id)


def list_articles(session: Session) -> list[Article]:
    stmt = select(Article).order_by(Article.id)
    return list(session.scalars(stmt).all())


def update_article(
    session: Session, article_id: int, **kwargs: str
) -> Optional[Article]:
    article = session.get(Article, article_id)
    if article is None:
        return None
    for key, value in kwargs.items():
        setattr(article, key, value)
    session.commit()
    session.refresh(article)
    return article


def delete_article(session: Session, article_id: int) -> bool:
    article = session.get(Article, article_id)
    if article is None:
        return False
    session.delete(article)
    session.commit()
    return True
```

### Existing Test Suite (MUST CONTINUE TO PASS)

```python
# test_articles.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session

# Import everything from article_system
from article_system import (
    Base, Article, SessionLocal,
    create_article, get_article, list_articles, update_article, delete_article,
)


@pytest.fixture
def session():
    """Fresh in-memory database for each test."""
    test_engine = create_engine("sqlite:///:memory:", echo=False)
    Base.metadata.create_all(test_engine)
    TestSession = sessionmaker(bind=test_engine)
    with TestSession() as s:
        yield s


def test_create_article(session):
    article = create_article(session, "Test Title", "test-title", "Body text")
    assert article.id is not None
    assert article.title == "Test Title"
    assert article.slug == "test-title"
    assert article.body == "Body text"
    assert article.status == "draft"


def test_get_article(session):
    created = create_article(session, "Get Test", "get-test")
    found = get_article(session, created.id)
    assert found is not None
    assert found.title == "Get Test"


def test_get_nonexistent_article(session):
    found = get_article(session, 9999)
    assert found is None


def test_list_articles(session):
    create_article(session, "Article A", "slug-a")
    create_article(session, "Article B", "slug-b")
    articles = list_articles(session)
    assert len(articles) == 2


def test_update_article(session):
    article = create_article(session, "Old Title", "update-test")
    updated = update_article(session, article.id, title="New Title")
    assert updated is not None
    assert updated.title == "New Title"
    assert updated.slug == "update-test"


def test_delete_article(session):
    article = create_article(session, "Delete Me", "delete-test")
    assert delete_article(session, article.id) is True
    assert get_article(session, article.id) is None
```

Run the tests before starting. They should all pass:
```
pytest test_articles.py -v
```

---

## Extension A: Search & Filter

**Task:** Add a `search_articles` function with this signature:

```python
def search_articles(
    session: Session,
    title_contains: Optional[str] = None,
    status: Optional[str] = None,
) -> list[Article]:
    """
    Search articles with optional filters.
    - title_contains: case-insensitive substring match on title
    - status: exact match on status
    - If both are None, return all articles ordered by id.
    """
    ...
```

**Requirements:**
- Case-insensitive title search (searching "python" must match "Learning Python Basics")
- Both filters are optional and can be combined
- Results ordered by `id`
- Existing tests must still pass

**New Tests for Extension A:**

```python
def test_search_by_title(session):
    create_article(session, "Learning Python", "python-101")
    create_article(session, "Advanced SQL", "sql-201")
    create_article(session, "Python for Backend", "python-backend")

    results = search_articles(session, title_contains="python")
    assert len(results) == 2
    assert all("python" in a.title.lower() for a in results)


def test_search_by_status(session):
    a1 = create_article(session, "Draft One", "draft-1")
    a2 = create_article(session, "Draft Two", "draft-2")
    update_article(session, a1.id, status="published")

    drafts = search_articles(session, status="draft")
    assert len(drafts) == 1
    assert drafts[0].slug == "draft-2"


def test_search_combined_filters(session):
    create_article(session, "Python Draft", "py-draft")
    create_article(session, "Python Published", "py-pub")
    update_article(
        session,
        get_article(session, 2).id,  # "Python Published"
        status="published",
    )
    create_article(session, "SQL Draft", "sql-draft")

    results = search_articles(session, title_contains="python", status="draft")
    assert len(results) == 1
    assert results[0].slug == "py-draft"


def test_search_no_filters_returns_all(session):
    create_article(session, "One", "one")
    create_article(session, "Two", "two")
    results = search_articles(session)
    assert len(results) == 2
```

---

## Extension B: Timestamps

**Task:** Add a `created_at` column to the `Article` model.

**Requirements:**
- Must use `server_default` — the database sets the timestamp, not Python
- Must be `Optional[datetime]` type (nullable in case of backfill scenarios)
- Existing tests must still pass **without modification** (no test passes `created_at`)
- After creation and refresh, `created_at` must be populated

**New Tests for Extension B:**

```python
from datetime import datetime


def test_created_at_populated(session):
    article = create_article(session, "Timestamped", "ts-test")
    assert article.created_at is not None
    assert isinstance(article.created_at, datetime)


def test_created_at_not_set_before_commit(session):
    """Verify you understand when server_default applies."""
    article = Article(title="Test", slug="test-pre-commit")
    assert article.created_at is None  # server_default: not set in Python yet
```

---

## Extension C: Status Machine

**Task:** Add a `publish_article` function that enforces a state transition rule: only articles with `status="draft"` can be published.

**Requirements:**

1. Define two custom exception classes (connect to Week 1, Lecture 2):
   ```python
   class ArticleNotFoundError(Exception):
       """Raised when an article ID does not exist."""
       ...

   class InvalidStatusTransitionError(Exception):
       """Raised when a status change violates allowed transitions."""
       ...
   ```

2. Implement:
   ```python
   def publish_article(session: Session, article_id: int) -> Article:
       """
       Transition article from 'draft' to 'published'.
       Raises ArticleNotFoundError if article does not exist.
       Raises InvalidStatusTransitionError if article is not 'draft'.
       """
       ...
   ```

3. The function must set `status = "published"`, commit, refresh, and return the article.

**New Tests for Extension C:**

```python
def test_publish_draft_article(session):
    article = create_article(session, "My Draft", "pub-test")
    assert article.status == "draft"

    published = publish_article(session, article.id)
    assert published.status == "published"


def test_publish_nonexistent_article(session):
    with pytest.raises(ArticleNotFoundError):
        publish_article(session, 9999)


def test_publish_already_published(session):
    article = create_article(session, "Already Done", "already-pub")
    publish_article(session, article.id)

    with pytest.raises(InvalidStatusTransitionError):
        publish_article(session, article.id)


def test_publish_does_not_corrupt_other_fields(session):
    article = create_article(session, "Keep My Data", "no-corrupt", "Important body")
    published = publish_article(session, article.id)
    assert published.title == "Keep My Data"
    assert published.body == "Important body"
    assert published.slug == "no-corrupt"
```

---

# Level 2.5 Solutions

## Solution A: Search & Filter

```python
def search_articles(
    session: Session,
    title_contains: Optional[str] = None,
    status: Optional[str] = None,
) -> list[Article]:
    stmt = select(Article)

    if title_contains is not None:
        # ilike = case-Insensitive LIKE. The % are SQL wildcards.
        stmt = stmt.where(Article.title.ilike(f"%{title_contains}%"))

    if status is not None:
        stmt = stmt.where(Article.status == status)

    stmt = stmt.order_by(Article.id)
    return list(session.scalars(stmt).all())
```

**Why this works:**
- `Article.title.ilike(...)` generates `WHERE title ILIKE '%python%'` — a case-insensitive pattern match. This is the SQLAlchemy equivalent of `LOWER(title) LIKE LOWER(...)`.
- Filters are **conditionally** chained onto the `select()` statement. This is the power of SQLAlchemy's composable query building — you construct the statement piece by piece in Python, and it assembles the final SQL for you.
- With no filters, the statement is just `SELECT ... ORDER BY id` — returning all articles.

---

## Solution B: Timestamps

**Model change:**
```python
from datetime import datetime
from sqlalchemy import DateTime, func

class Article(Base):
    __tablename__ = "articles"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    slug: Mapped[str] = mapped_column(String(200), unique=True)
    body: Mapped[Optional[str]] = mapped_column(Text)
    status: Mapped[str] = mapped_column(String(20), default="draft")
    # NEW: server-side timestamp
    created_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime, server_default=func.now()
    )

    def __repr__(self) -> str:
        return f"Article(id={self.id!r}, title={self.title!r}, status={self.status!r})"
```

**Why this works and existing tests still pass:**
- `server_default=func.now()` means the database adds a `DEFAULT CURRENT_TIMESTAMP` to the column. No Python code needs to supply a value.
- `Mapped[Optional[datetime]]` means the column is nullable, so `create_all()` doesn't add `NOT NULL`. This is important: `server_default` populates it on INSERT, but the Python object sees `None` until `refresh()`.
- Existing tests never assert on `created_at`. The `create_article` function already calls `session.refresh(article)`, which loads the server-generated timestamp. No code changes needed beyond the model.
- `test_created_at_not_set_before_commit` passes because `server_default` is a DB-side mechanism — the Python constructor sets `created_at = None` (Exercise 1.3's lesson in action).

---

## Solution C: Status Machine

**Exceptions:**
```python
class ArticleNotFoundError(Exception):
    """Raised when an article ID does not exist."""

    def __init__(self, article_id: int) -> None:
        self.article_id = article_id
        super().__init__(f"Article with id={article_id} not found")


class InvalidStatusTransitionError(Exception):
    """Raised when a status change violates allowed transitions."""

    def __init__(self, current_status: str, target_status: str) -> None:
        self.current_status = current_status
        self.target_status = target_status
        super().__init__(
            f"Cannot transition from '{current_status}' to '{target_status}'"
        )
```

**Function:**
```python
def publish_article(session: Session, article_id: int) -> Article:
    article = session.get(Article, article_id)

    if article is None:
        raise ArticleNotFoundError(article_id)

    if article.status != "draft":
        raise InvalidStatusTransitionError(article.status, "published")

    article.status = "published"
    session.commit()
    session.refresh(article)
    return article
```

**Why this works:**
- Validation happens **before** modification (the lesson from Exercise 2.2). If the article doesn't exist or isn't in "draft" status, no attribute is changed, no commit is attempted.
- Custom exceptions carry **context** (which article, what transition was attempted). This connects directly to Week 1, Lecture 2: generic exceptions provide no debugging information, custom exceptions tell you exactly what went wrong.
- `session.refresh(article)` after commit ensures the returned object has populated attributes (the lesson from Exercise 2.1).
- `test_publish_does_not_corrupt_other_fields` passes because we only modified `status`; the Session's dirty tracking only generates `UPDATE articles SET status='published' WHERE id=...`. Other columns are untouched.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## Event Ticketing System

**Scenario:** You're building the backend for a small event ticketing platform. Events have limited capacity, and ticket sales must be reliable — no overselling, no orphaned tickets, no money-related ambiguity.

**Your task:** Implement the models, exceptions, and repository functions. The test file below is **read-only** — you must make every test pass without modifying it.

### Provided Test File (DO NOT MODIFY)

```python
# test_ticketing.py
import pytest
from sqlalchemy import create_engine, select
from sqlalchemy.orm import sessionmaker, Session

# --- Students must implement these imports ---
from ticketing.models import Base, Event, Ticket
from ticketing.exceptions import SoldOutError, EventNotFoundError, TicketNotFoundError
from ticketing.repository import create_event, purchase_ticket, transfer_ticket


@pytest.fixture
def session():
    test_engine = create_engine("sqlite:///:memory:", echo=False)
    Base.metadata.create_all(test_engine)
    TestSession = sessionmaker(bind=test_engine)
    with TestSession() as s:
        yield s


# ===== 3a: Core Models & Basic Operations =====

class TestCoreModels:
    def test_create_event(self, session):
        event = create_event(session, name="Python Conf", capacity=100)
        assert event.id is not None
        assert event.name == "Python Conf"
        assert event.capacity == 100

    def test_purchase_ticket(self, session):
        event = create_event(session, name="Concert", capacity=50)
        ticket = purchase_ticket(session, event_id=event.id, holder_name="Alice")
        assert ticket.id is not None
        assert ticket.holder_name == "Alice"
        assert ticket.event_id == event.id

    def test_purchase_for_nonexistent_event(self, session):
        with pytest.raises(EventNotFoundError):
            purchase_ticket(session, event_id=9999, holder_name="Ghost")

    def test_multiple_tickets_same_event(self, session):
        event = create_event(session, name="Workshop", capacity=10)
        t1 = purchase_ticket(session, event.id, "Alice")
        t2 = purchase_ticket(session, event.id, "Bob")
        assert t1.id != t2.id
        assert t1.holder_name == "Alice"
        assert t2.holder_name == "Bob"


# ===== 3b: Capacity Enforcement =====

class TestCapacity:
    def test_cannot_exceed_capacity(self, session):
        event = create_event(session, name="Small Show", capacity=2)
        purchase_ticket(session, event.id, "Alice")
        purchase_ticket(session, event.id, "Bob")
        with pytest.raises(SoldOutError):
            purchase_ticket(session, event.id, "Charlie")

    def test_exact_capacity_boundary(self, session):
        event = create_event(session, name="Trio Show", capacity=3)
        purchase_ticket(session, event.id, "Alice")
        purchase_ticket(session, event.id, "Bob")
        ticket = purchase_ticket(session, event.id, "Charlie")
        assert ticket is not None
        with pytest.raises(SoldOutError):
            purchase_ticket(session, event.id, "Diana")

    def test_sold_out_does_not_corrupt_existing_data(self, session):
        event = create_event(session, name="Tiny Show", capacity=1)
        purchase_ticket(session, event.id, "Alice")
        with pytest.raises(SoldOutError):
            purchase_ticket(session, event.id, "Bob")
        # Alice's ticket must still exist and be correct
        stmt = select(Ticket).where(Ticket.event_id == event.id)
        tickets = list(session.scalars(stmt).all())
        assert len(tickets) == 1
        assert tickets[0].holder_name == "Alice"

    def test_different_events_independent_capacity(self, session):
        event_a = create_event(session, name="Show A", capacity=1)
        event_b = create_event(session, name="Show B", capacity=1)
        purchase_ticket(session, event_a.id, "Alice")
        # event_a is sold out, but event_b should still work
        ticket = purchase_ticket(session, event_b.id, "Bob")
        assert ticket.holder_name == "Bob"


# ===== 3c: Ticket Transfer =====

class TestTransfer:
    def test_transfer_ticket(self, session):
        event = create_event(session, name="Gala", capacity=10)
        ticket = purchase_ticket(session, event.id, "Alice")
        transferred = transfer_ticket(session, ticket.id, new_holder="Bob")
        assert transferred.holder_name == "Bob"
        assert transferred.id == ticket.id  # Same ticket, different holder

    def test_transfer_nonexistent_ticket(self, session):
        with pytest.raises(TicketNotFoundError):
            transfer_ticket(session, ticket_id=9999, new_holder="Ghost")

    def test_transfer_preserves_event_association(self, session):
        event = create_event(session, name="Festival", capacity=5)
        ticket = purchase_ticket(session, event.id, "Alice")
        transferred = transfer_ticket(session, ticket.id, "Bob")
        assert transferred.event_id == event.id

    def test_transfer_does_not_free_capacity(self, session):
        event = create_event(session, name="Limited", capacity=2)
        t1 = purchase_ticket(session, event.id, "Alice")
        purchase_ticket(session, event.id, "Bob")
        # Sold out
        with pytest.raises(SoldOutError):
            purchase_ticket(session, event.id, "Charlie")
        # Transfer Alice's ticket to Charlie
        transfer_ticket(session, t1.id, "Charlie")
        # Still sold out — transfer doesn't create new capacity
        with pytest.raises(SoldOutError):
            purchase_ticket(session, event.id, "Diana")
```

---

### 3a: Implement Core Models and Basic Operations

**Your file structure:**

```
ticketing/
├── __init__.py
├── models.py        # Event and Ticket models
├── exceptions.py    # SoldOutError, EventNotFoundError, TicketNotFoundError
└── repository.py    # create_event, purchase_ticket, transfer_ticket
```

**Hints:**
- `Event` needs: `id`, `name` (String), `capacity` (int)
- `Ticket` needs: `id`, `event_id` (int), `holder_name` (String)
- No SQLAlchemy `relationship()` needed — store `event_id` as a plain `Mapped[int]`
- Your exceptions should carry context (which event ID, which ticket ID). Think back to Week 1.

**Target:** Make all `TestCoreModels` tests pass.

---

### 3b: Add Capacity Enforcement

**Constraint:** The `purchase_ticket` function must refuse to sell more tickets than the event's capacity. The check and the insert must be **atomic** — there must be no window where the count could change between checking and inserting.

**Hints:**
- You need to count existing tickets for the event before creating a new one
- Both the count query and the INSERT must be in the same transaction (single commit)
- Think about what happens if `SoldOutError` is raised — does the session need cleanup?

**Target:** Make all `TestCapacity` tests pass (plus all `TestCoreModels` tests still pass).

---

### 3c: Add Ticket Transfer

**Constraint:** Transferring a ticket changes the holder but does NOT free up capacity. The ticket itself persists — only the holder changes.

**Hints:**
- This is conceptually simple: load the ticket, change `holder_name`, commit
- But you must handle the error cases (nonexistent ticket)
- Think about: does transfer require flushing intermediate state? Or is a simple attribute change + commit sufficient?

**Target:** Make ALL tests pass (TestCoreModels + TestCapacity + TestTransfer).

---

# Level 3 Solutions

## exceptions.py

```python
class TicketingError(Exception):
    """Base exception for the ticketing system."""


class EventNotFoundError(TicketingError):
    def __init__(self, event_id: int) -> None:
        self.event_id = event_id
        super().__init__(f"Event with id={event_id} not found")


class SoldOutError(TicketingError):
    def __init__(self, event_id: int, event_name: str) -> None:
        self.event_id = event_id
        super().__init__(f"Event '{event_name}' (id={event_id}) is sold out")


class TicketNotFoundError(TicketingError):
    def __init__(self, ticket_id: int) -> None:
        self.ticket_id = ticket_id
        super().__init__(f"Ticket with id={ticket_id} not found")
```

**Why a base class?**
This connects to Week 1, Lecture 2: exception hierarchies. All ticketing exceptions inherit from `TicketingError`, allowing callers to catch all ticketing errors with a single `except TicketingError` or specific subclasses for granular handling.

## models.py

```python
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Event(Base):
    __tablename__ = "events"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    capacity: Mapped[int] = mapped_column()

    def __repr__(self) -> str:
        return f"Event(id={self.id!r}, name={self.name!r}, capacity={self.capacity!r})"


class Ticket(Base):
    __tablename__ = "tickets"
    id: Mapped[int] = mapped_column(primary_key=True)
    event_id: Mapped[int] = mapped_column()
    holder_name: Mapped[str] = mapped_column(String(200))

    def __repr__(self) -> str:
        return f"Ticket(id={self.id!r}, event_id={self.event_id!r}, holder={self.holder_name!r})"
```

**Note:** `event_id` is a plain `Mapped[int]` — no `ForeignKey` declaration. Foreign keys and `relationship()` are Lecture 2 topics. For this exercise, the link between tickets and events is maintained by the repository logic, not by database-level referential integrity.

## repository.py

```python
from sqlalchemy import select, func
from sqlalchemy.orm import Session

from ticketing.models import Event, Ticket
from ticketing.exceptions import EventNotFoundError, SoldOutError, TicketNotFoundError


def create_event(session: Session, name: str, capacity: int) -> Event:
    event = Event(name=name, capacity=capacity)
    session.add(event)
    session.commit()
    session.refresh(event)
    return event


def purchase_ticket(session: Session, event_id: int, holder_name: str) -> Ticket:
    # 1. Load the event
    event = session.get(Event, event_id)
    if event is None:
        raise EventNotFoundError(event_id)

    # 2. Count existing tickets (within the same transaction)
    stmt = select(func.count()).select_from(Ticket).where(
        Ticket.event_id == event_id
    )
    ticket_count = session.scalar(stmt)

    # 3. Check capacity
    if ticket_count >= event.capacity:
        raise SoldOutError(event_id, event.name)

    # 4. Create the ticket (same transaction as the count)
    ticket = Ticket(event_id=event_id, holder_name=holder_name)
    session.add(ticket)
    session.commit()
    session.refresh(ticket)
    return ticket


def transfer_ticket(session: Session, ticket_id: int, new_holder: str) -> Ticket:
    ticket = session.get(Ticket, ticket_id)
    if ticket is None:
        raise TicketNotFoundError(ticket_id)

    ticket.holder_name = new_holder
    session.commit()
    session.refresh(ticket)
    return ticket
```

**Why this design meets the constraints:**

**Atomicity of capacity check:** The `SELECT COUNT(*)` and the `INSERT` happen within the same session transaction. The count is accurate at the time of the check. If we were to raise `SoldOutError`, no SQL for the INSERT was sent (we raised before `session.add`), so no cleanup is needed. This is the same "validate before modify" pattern from Exercise 2.2.

**`SoldOutError` doesn't corrupt data:** Because the error is raised *before* `session.add(ticket)`, the session has no pending changes. No rollback needed. Existing tickets are untouched.

**Transfer doesn't affect capacity:** Transfer modifies `holder_name` on an existing ticket. It doesn't delete and re-create. The ticket count for the event doesn't change. The test `test_transfer_does_not_free_capacity` verifies this: after transfer, `SELECT COUNT(*)` still returns the same number.

**Limitation:** In a concurrent multi-user environment (multiple simultaneous purchases), this approach has a race condition: two requests could both count N-1 tickets, both pass the capacity check, and both insert — exceeding capacity. The fix (optimistic locking or `SELECT ... FOR UPDATE`) is covered in Week 7, Lecture 2. For this exercise with single-session tests, the solution is correct.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## Exercise: When the ORM Works Against You

**Task:** Describe a scenario where using the SQLAlchemy ORM to perform a task is significantly **worse** than using raw SQL. Write two code snippets:

1. The ORM approach (showing why it's problematic)
2. The raw SQL / SQLAlchemy Core approach (showing why it's better)

Explain the performance difference in terms of what SQL is generated and how many round-trips to the database occur.

**Hint:** Think about operations that involve many rows. The Session tracks every object individually. What happens when "every object individually" means thousands of objects?

---

# Level 4 Solution

**Scenario: Bulk Insert of 10,000 Rows**

### ORM Approach (Slow)

```python
import time
from sqlalchemy import create_engine, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session

engine = create_engine("sqlite:///bulk_test.db", echo=False)

class Base(DeclarativeBase):
    pass

class SensorReading(Base):
    __tablename__ = "readings"
    id: Mapped[int] = mapped_column(primary_key=True)
    sensor_name: Mapped[str] = mapped_column(String(50))
    value: Mapped[int] = mapped_column()

Base.metadata.create_all(engine)

# --- ORM bulk insert ---
start = time.time()
with Session(engine) as session:
    for i in range(10_000):
        reading = SensorReading(sensor_name=f"sensor_{i % 100}", value=i)
        session.add(reading)
    session.commit()
elapsed_orm = time.time() - start
print(f"ORM: {elapsed_orm:.2f}s")
```

**What happens under the hood:**
1. Python creates 10,000 `SensorReading` objects in memory — each tracked individually by the Session
2. On `commit()`, the Session iterates over all 10,000 pending objects
3. For each object, it generates and executes a separate `INSERT INTO readings ...` statement
4. That's **10,000 individual SQL statements** sent to the database

The Session's identity map, dirty tracking, and object state management add overhead per object that's unnecessary when you're simply inserting rows with no need to track them afterward.

### SQLAlchemy Core Approach (Fast)

```python
from sqlalchemy import text

# --- Core bulk insert ---
start = time.time()
data = [
    {"sensor_name": f"sensor_{i % 100}", "value": i}
    for i in range(10_000)
]
with engine.connect() as conn:
    conn.execute(
        text("INSERT INTO readings (sensor_name, value) VALUES (:sensor_name, :value)"),
        data,
    )
    conn.commit()
elapsed_core = time.time() - start
print(f"Core: {elapsed_core:.2f}s")
print(f"Speedup: {elapsed_orm / elapsed_core:.1f}x")
```

**What happens under the hood:**
1. Python creates a list of 10,000 lightweight dictionaries (no ORM object overhead)
2. `conn.execute()` with a list of parameters uses the DB driver's `executemany` — which can batch all 10,000 inserts into far fewer round-trips
3. No identity map, no dirty tracking, no object state management

**Typical result:** The Core approach is **5-20x faster** depending on the database driver and row count.

### When to Choose Raw SQL / Core Over ORM

```
USE THE ORM WHEN:
├─ You need to read, modify, and write back individual objects
├─ You need to track changes (dirty detection)
├─ You're working within a web request (load user, modify, save)
└─ Object identity matters (same row = same Python object)

USE RAW SQL / CORE WHEN:
├─ Bulk inserting thousands+ of rows (no need to track each)
├─ Complex analytical queries (window functions, recursive CTEs, pivots)
├─ Database-specific features the ORM can't express
├─ One-off data migrations or ETL scripts
└─ Performance-critical paths where you need control over exact SQL
```

**The Key Insight:**
The ORM's power — automatic change tracking, identity map, lifecycle management — is also its cost. Each managed object carries overhead. For operations that don't need management (bulk inserts, read-only analytics, mass updates), bypassing the ORM and using Core or raw SQL can be dramatically faster. Knowing when NOT to use a tool is as important as knowing how to use it.

---

# LEVEL 5: THE TEACH BACK

## Scenario

A junior developer on your team submits the following code in a pull request and writes in the PR description:

> "I use `session.commit()` everywhere because that's the one that saves data. I tried `session.flush()` once but it does the same thing — both save your data to the database. So I just use commit() and ignore flush(). Also, in the `register_user` function, I added error handling — if the username is taken, I catch the error and return None. Clean and simple."

```python
def register_user(session, username, email):
    try:
        user = User(username=username, email=email)
        session.add(user)
        session.commit()
        return user
    except IntegrityError:
        return None

def create_user_with_profile(session, username, email, bio):
    user = register_user(session, username, email)
    if user is None:
        return None

    profile = Profile(user_id=user.id, bio=bio)
    session.add(profile)
    session.commit()
    return user
```

**Your task:** Write a code review response. Specifically:

1. **Address the `flush() == commit()` misconception.** Give a concrete example where using `flush()` instead of `commit()` is necessary and where using `commit()` would cause a bug. (Hint: `create_user_with_profile` needs an intermediate ID.)

2. **Address the error handling.** Explain why `except IntegrityError: return None` poisons the session and will cause `create_user_with_profile` to fail on the very next line. What exception will the next `session.commit()` raise?

3. **Rewrite both functions** so they work correctly together.

---

# Level 5 Solution

## 1. Why `flush()` ≠ `commit()`

"They are not the same. Here's the difference that matters for your code:

`flush()` sends SQL to the database **within an open transaction**. The database processes the INSERT and assigns an auto-increment ID. But the transaction is still open — the changes are provisional. If anything goes wrong afterward, you can `rollback()` and undo everything, including the INSERT.

`commit()` sends SQL **and** finalizes the transaction. The changes are permanent. There is no undo.

In your `create_user_with_profile`, you need the user's `id` to create the profile (because `Profile` has `user_id`). But you also need both the user and the profile to be created **atomically** — if the profile creation fails, the user should not exist either.

`flush()` solves exactly this: get the ID without committing.

If you use `commit()` after creating the user, the user is permanent. If the profile creation then fails, you have an orphaned user — exactly the bug from Exercise 2.2."

## 2. Why the Error Handling is Dangerous

"Your `except IntegrityError: return None` catches the Python exception, but it does NOT fix the Session. After an `IntegrityError`, the database transaction is in an error state. The Session records this and requires an explicit `rollback()` before any further SQL can execute.

In `create_user_with_profile`:
1. If `register_user` catches `IntegrityError` without rollback and returns `None`
2. The caller checks `if user is None: return None` — fine, it returns early
3. But what if it DIDN'T return early? The very next `session.commit()` would raise `PendingRollbackError`

Worse: if another part of your code reuses this session later, it's poisoned. Every operation will fail with `PendingRollbackError` until someone calls `session.rollback()`. This is a ticking time bomb in a web application where sessions might be reused across requests."

## 3. Correct Implementation

```python
def create_user_with_profile(
    session: Session, username: str, email: str, bio: str
) -> Optional[User]:
    """
    Create a user and their profile atomically.
    Returns the user, or None if username/email is taken.
    """
    try:
        # Create user — don't commit yet
        user = User(username=username, email=email)
        session.add(user)
        session.flush()  # ← Get user.id WITHOUT committing

        # Create profile using the flushed ID
        profile = Profile(user_id=user.id, bio=bio)
        session.add(profile)

        # Commit BOTH in a single transaction
        session.commit()
        session.refresh(user)
        return user

    except IntegrityError:
        session.rollback()  # ← MUST rollback to clear the poisoned transaction
        return None
```

**What this fixes:**

- **`flush()` instead of `commit()`:** The user gets an `id` from the database (needed for `Profile.user_id`), but the transaction stays open. If the profile INSERT fails, the user INSERT is also rolled back.
- **`session.rollback()` in the except block:** Clears the poisoned transaction state. The session is now usable again for future operations.
- **Single `commit()`:** Both the user and profile are in the same transaction. Either both exist or neither does. Atomicity guaranteed.

**The rule of thumb:**
Use `flush()` when you need intermediate database-generated values (IDs, defaults) but aren't done with your logical operation yet. Use `commit()` only when the entire logical unit of work is complete. And always `rollback()` after catching a database error — the Python `except` handles the exception; the `rollback()` heals the Session.