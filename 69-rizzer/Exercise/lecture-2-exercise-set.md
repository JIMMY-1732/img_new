# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "These are just Python syntax features I need to memorize. Dataclasses are like regular classes, context managers are just try/finally, decorators are magic @symbols, and exceptions are things I catch with `except Exception`."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | Dataclasses validate types at runtime | Type hints are not enforced; dataclasses just use them to identify fields |
| 2 | Mutable defaults in dataclasses are safe | Default `[]` or `{}` are shared across ALL instances |
| 3 | Decorators are special syntax that "does something" | `@decorator` is literally just `func = decorator(func)` |
| 4 | Decorators run when the function is called | Decorators run at **definition time**, wrappers run at call time |
| 5 | Context managers are just for files | Any resource needing cleanup can/should use context managers |
| 6 | `except Exception` is good defensive coding | It swallows bugs and makes debugging impossible |
| 7 | Custom exceptions are overkill | Generic errors provide no context for handling or debugging |

**The Prediction Gap:** When given code with a mutable default in a dataclass, students expect each instance to have its own independent list. The actual behavior is that ALL instances share the SAME list, causing mysterious "data corruption" bugs.

---

## The Main Purpose (Essence)

**Core Insight:** These four patterns solve the same meta-problem: **separating WHAT from HOW**—separating the core logic from the surrounding infrastructure (boilerplate, cleanup, cross-cutting concerns, error context).

**The Problems Being Solved:**

| Pattern | Without It | With It |
|---------|-----------|---------|
| Dataclasses | Write `__init__`, `__repr__`, `__eq__` for every data class | Declare fields; methods generated automatically |
| Context Managers | Remember to call cleanup in every code path | Cleanup guaranteed regardless of how block exits |
| Decorators | Copy-paste logging/timing/retry into every function | Define behavior once, apply with single line |
| Custom Exceptions | Debug with "Something went wrong" | Debug with structured context: what, where, why |

**The Key Distinction:**
- **Dataclass** means "generate boilerplate methods from field declarations"
- **Pydantic BaseModel** means "validate data at runtime" (Week 2)
- The syntax looks similar, but the purpose is fundamentally different.

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|--------------|---------------------|-----------------|------------|
| "Dataclass enforces types" | `User(id="abc")` raises error | Creates User with string id | Type hints are metadata only |
| "Default `[]` creates new list" | Two users have independent tags | Both users share same list | Default evaluated once at class definition |
| "Decorator runs on function call" | Print in decorator shows on each call | Print shows once at import time | Decorator runs at definition, wrapper runs at call |
| "`@wraps` is optional" | Decorated function works fine | `__name__`, `__doc__` are wrong | Wrapper replaces original; metadata lost |
| "`except Exception` is safe" | Catches all errors gracefully | Swallows `KeyError` from typo, hides bug | Too broad; catches programming errors |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Shared Secret (Dataclass Mutable Default)

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    permissions: list[str] = []

# Create two separate users
alice = User(name="alice")
bob = User(name="bob")

# Give Alice admin permissions
alice.permissions.append("admin")
alice.permissions.append("write")

# Check Bob's permissions
print(f"Alice permissions: {alice.permissions}")
print(f"Bob permissions: {bob.permissions}")
print(f"Same list? {alice.permissions is bob.permissions}")
```

**Questions:**

1. **Predict the output.** Specifically:
   - What permissions does Alice have?
   - What permissions does Bob have?
   - What does `alice.permissions is bob.permissions` return?

2. **Explain why.** A junior developer expected Bob to have no permissions since we never called `bob.permissions.append()`. Why does Bob have permissions he was never given?

3. **Mental Trace:** Walk through what happens when Python first encounters the `class User` definition. When is the empty list `[]` created? How many list objects exist after creating both users?

---

## Exercise 1.2: When Does This Print? (Decorator Timing)

```python
def register_endpoint(func):
    print(f"Registering endpoint: {func.__name__}")
    
    def wrapper(*args, **kwargs):
        print(f"Calling endpoint: {func.__name__}")
        return func(*args, **kwargs)
    
    return wrapper

print("=== Application Starting ===")

@register_endpoint
def get_users():
    return ["alice", "bob"]

@register_endpoint  
def get_orders():
    return ["order1", "order2"]

print("=== Application Ready ===")
print("=== Making First Request ===")

result = get_users()
print(f"Result: {result}")
```

**Questions:**

1. **Predict the exact output.** Write each line in the order it appears, including all "===" markers.

2. **Explain why.** A student expected "Registering endpoint: get_users" to print when `get_users()` is called. Why does it print earlier?

3. **Mental Trace:** The `@register_endpoint` syntax is equivalent to what explicit Python code? At what moment does `register_endpoint(get_users)` execute?

---

## Exercise 1.3: The Swallowed Bug (Exception Handling)

```python
def get_user_data(user_dict: dict, user_id: str) -> dict:
    try:
        user = user_dict[user_id]
        # Typo: 'emial' instead of 'email'
        return {"name": user["name"], "contact": user["emial"]}
    except Exception:
        return {"error": "User not found"}

users = {
    "u1": {"name": "Alice", "email": "alice@example.com"},
    "u2": {"name": "Bob", "email": "bob@example.com"},
}

# Test 1: Valid user
result1 = get_user_data(users, "u1")
print(f"User u1: {result1}")

# Test 2: Invalid user
result2 = get_user_data(users, "u999")
print(f"User u999: {result2}")
```

**Questions:**

1. **Predict the output** for both test cases.

2. **Explain the bug.** The function has a typo (`emial` instead of `email`). Why doesn't this cause a visible error? Why is this dangerous?

3. **Rewrite** the exception handling to catch ONLY the case where the user doesn't exist, while letting actual bugs (like typos) crash loudly.

---

## Exercise 1.4: Context Manager Guarantee

```python
class DatabaseConnection:
    def __init__(self, name: str):
        self.name = name
        self.connected = False
    
    def connect(self):
        print(f"[{self.name}] Connecting...")
        self.connected = True
    
    def disconnect(self):
        print(f"[{self.name}] Disconnecting...")
        self.connected = False
    
    def query(self, sql: str):
        if not self.connected:
            raise RuntimeError("Not connected!")
        if "DROP" in sql:
            raise ValueError("DROP not allowed!")
        print(f"[{self.name}] Executing: {sql}")
        return f"Results for: {sql}"

# Scenario A: Normal operation
print("=== Scenario A ===")
db_a = DatabaseConnection("DB-A")
db_a.connect()
try:
    result = db_a.query("SELECT * FROM users")
    print(result)
finally:
    db_a.disconnect()

# Scenario B: Error during query
print("\n=== Scenario B ===")
db_b = DatabaseConnection("DB-B")
db_b.connect()
try:
    result = db_b.query("DROP TABLE users")  # This will raise!
    print(result)
finally:
    db_b.disconnect()

print("\n=== After Both Scenarios ===")
print(f"DB-A connected: {db_a.connected}")
print(f"DB-B connected: {db_b.connected}")
```

**Questions:**

1. **Predict the output.** Include all print statements and any exception messages. Does Scenario B's error prevent the disconnect?

2. **Explain why** `finally` guarantees the disconnect happens even when `query()` raises an exception. What would happen WITHOUT the try/finally?

3. **Observation:** This try/finally pattern is exactly what context managers automate. Rewrite Scenario A using a context manager class with `__enter__` and `__exit__`.

---

# Level 1 Solutions

## Solution 1.1: The Shared Secret

1. **Predict the output**
* Alice permissions: ['admin', 'write']
* Bob permissions: ['admin', 'write']
* Same list? True

2. **Explain why:**
* In Python, default arguments are evaluated only once, at the moment the class (or function) is defined, not every time you create a new instance.
* When Python reads the class User definition, it creates one single ```list []``` in memory.
* Both alice and bob are assigned a reference to this same list object.
* When you modify Alice's list, you are modifying the shared list that Bob also looks at.

3: **Mental Trace:**
```
1. Python reads "class User:"
2. Python evaluates "permissions: list[str] = []"
   → Creates ONE list object: list_object_001
3. This list_object_001 is stored as the default for 'permissions'
4. User("alice") created → alice.permissions = list_object_001
5. User("bob") created → bob.permissions = list_object_001 (SAME object!)
6. alice.permissions.append("admin") modifies list_object_001
7. bob.permissions IS list_object_001, so Bob "sees" the change
```

**The Key Insight:**
Default values are evaluated at **definition time**, not at **instantiation time**. For mutable objects, this means all instances share the same object.

**The Fix:**
```python
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    permissions: list[str] = field(default_factory=list)  # New list per instance
```

---

## Solution 1.2: When Does This Print?

1. **Predict the exact output**
```
=== Application Starting ===
Registering endpoint: get_users
Registering endpoint: get_orders
=== Application Ready ===
=== Making First Request ===
Calling endpoint: get_users
Result: ['alice', 'bob']
```

2. **Explain why**
* The **naive model** predicts "Registering" prints when we call `get_users()`. But `@register_endpoint` is syntactic sugar for:
```python
get_users = register_endpoint(get_users)
```

* This assignment happens **immediately after the function is defined**, not when called. The decorator function runs at definition time; the wrapper function runs at call time.

3. **Mental Trace:**
- The syntax @register_endpoint is syntactic sugar. Python translates your code to this:
```Python
def get_users():
    return ["alice", "bob"]

# This executes IMMEDIATELY when the file is read:
get_users = register_endpoint(get_users) 
```

**Flow:**
```
1. print("=== Application Starting ===")           → prints
2. def get_users(): ...                            → function object created
3. get_users = register_endpoint(get_users)        → DECORATOR RUNS NOW
   → print("Registering endpoint: get_users")      → prints
   → returns wrapper function
4. def get_orders(): ...                           → function object created  
5. get_orders = register_endpoint(get_orders)      → DECORATOR RUNS NOW
   → print("Registering endpoint: get_orders")     → prints
6. print("=== Application Ready ===")              → prints
7. print("=== Making First Request ===")           → prints
8. get_users()                                     → calls WRAPPER
   → print("Calling endpoint: get_users")          → prints
   → calls original get_users()
   → returns ["alice", "bob"]
9. print(f"Result: {result}")                      → prints
```

**The Key Insight:**
`@decorator` runs the decorator function **at import/definition time**. This is why FastAPI's `@app.get("/users")` registers routes when your module loads, not when requests arrive.

---

## Solution 1.3: The Swallowed Bug

1. **Predict the output**
```
User u1: {'error': 'User not found'}
User u999: {'error': 'User not found'}
```

2. **Explain the bug**

- Both cases return the error message! For `u1`, the user EXISTS, but `user["emial"]` raises `KeyError` due to the typo. `except Exception` catches this `KeyError` and returns "User not found"—completely hiding the bug.

- The dangerous part: The code appears to work (no crashes), but it's silently returning wrong results for valid users. This bug could go unnoticed for months.

3. **Rewrite**
```python
def get_user_data(user_dict: dict, user_id: str) -> dict:
    try:
        user = user_dict[user_id]
    except KeyError:
        return {"error": "User not found"}
    
    # Outside try/except: typos here will crash (good!)
    return {"name": user["name"], "contact": user["email"]}
```

Or with more specific handling:
```python
def get_user_data(user_dict: dict, user_id: str) -> dict:
    if user_id not in user_dict:
        return {"error": "User not found"}
    
    user = user_dict[user_id]
    return {"name": user["name"], "contact": user["email"]}
```

**The Key Insight:**
`except Exception` is almost always wrong. Catch specific exceptions for expected error conditions. Let unexpected errors (bugs) crash loudly so you find them.

---

## Solution 1.4: Context Manager Guarantee

1. **Correct Output:**
```
=== Scenario A ===
[DB-A] Connecting...
[DB-A] Executing: SELECT * FROM users
Results for: SELECT * FROM users
[DB-A] Disconnecting...

=== Scenario B ===
[DB-B] Connecting...
[DB-B] Disconnecting...
Traceback (most recent call last):
  ...
ValueError: DROP not allowed!

=== After Both Scenarios ===
```
(Note: The "After Both Scenarios" section doesn't print because the exception propagates. If we wrap Scenario B in another try/except, we'd see both are disconnected.)

2. **Explain why:**
- `finally` blocks **always execute**, whether the try block succeeds, raises an exception, or even contains a `return` statement. This is why `finally` is perfect for cleanup.

Without try/finally:
```python
db_b.connect()
result = db_b.query("DROP TABLE users")  # Raises!
db_b.disconnect()  # NEVER RUNS - connection leaked!
```

3. **Rewrite with Context Manager**
```python
class DatabaseConnection:
    # ... __init__, connect, disconnect, query as before ...
    
    def __enter__(self):
        self.connect()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.disconnect()
        return False  # Don't suppress exceptions

# Usage
with DatabaseConnection("DB-A") as db:
    result = db.query("SELECT * FROM users")
    print(result)
# disconnect() automatically called!
```

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Decorator That Forgot

**Context:** A developer wrote a timing decorator for API endpoints. The code runs, but something is wrong when debugging.

```python
import time
from typing import Callable, Any

def timed(func: Callable) -> Callable:
    """Decorator that logs execution time"""
    
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - start
        print(f"{func.__name__} took {elapsed:.3f}s")
        return result
    
    return wrapper

@timed
def fetch_user(user_id: int) -> dict:
    """Fetch user from database by ID.
    
    Args:
        user_id: The unique user identifier
        
    Returns:
        User data dictionary
    """
    time.sleep(0.1)  # Simulate database query
    return {"id": user_id, "name": "Alice"}

# Test the decorator
result = fetch_user(42)
print(f"Result: {result}")

# Debugging session
print(f"\nFunction name: {fetch_user.__name__}")
print(f"Function doc: {fetch_user.__doc__}")
print(f"Help output:")
help(fetch_user)
```

---

**Task 2a: Identify the Flaw**

This decorator works functionally—it correctly times the function. However, there is a flaw that causes problems during debugging and documentation.

1. Run the code. What is wrong with the function name and docstring?
2. Why is this problematic for a production codebase?

Do NOT fix the code yet. Just identify the problem.

---

**Task 2b: Explain the Mechanism**

Explain what happens when `@timed` is applied:

1. What object does `fetch_user` refer to after decoration?
2. Where did the original function's `__name__` and `__doc__` go?
3. Why does `help(fetch_user)` show useless information?

---

**Task 2c: Instrument to Prove Diagnosis**

Add print statements to demonstrate that `fetch_user` is actually the `wrapper` function:

```python
# Add instrumentation to prove the identity problem
print(f"fetch_user is wrapper: {???}")
print(f"Original function name preserved: {???}")
```

Show output proving your diagnosis.

---

**Task 2d: Fix the Code**

Fix the decorator so that:
- Timing still works correctly
- `fetch_user.__name__` returns `"fetch_user"`
- `fetch_user.__doc__` returns the original docstring
- `help(fetch_user)` shows useful documentation

Hint: Use `functools.wraps`.

---

**Task 2e: Write Verification Test**

Write a test that:
- FAILS on the original (broken) decorator
- PASSES on your fixed decorator

```python
def test_metadata_preserved():
    """
    This test fails on the unfixed decorator, passes on fixed.
    """
    @timed
    def sample_function():
        """Sample docstring."""
        pass
    
    # Your assertions here
    assert sample_function.__name__ == ???
    assert sample_function.__doc__ == ???
```

---

## Exercise 2.2: The Exception That Hides Everything

**Context:** A developer wrote a data processing function with "robust" error handling. It never crashes, but it also never works correctly.

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class ProcessingResult:
    success: bool
    data: Optional[dict] = None
    error: Optional[str] = None

def process_user_data(raw_data: dict) -> ProcessingResult:
    """Process raw user data into clean format."""
    try:
        # Extract required fields
        user_id = raw_data["user_id"]
        email = raw_data["email"]
        
        # Validate email (intentional bug: 'emal' typo)
        if "@" not in raw_data["emal"]:
            return ProcessingResult(success=False, error="Invalid email")
        
        # Process age (might not exist)
        age = raw_data.get("age", 0)
        if age < 0:
            return ProcessingResult(success=False, error="Invalid age")
        
        # Build clean data
        clean_data = {
            "id": user_id,
            "email": email.lower().strip(),
            "age": age,
            "is_adult": age >= 18
        }
        
        return ProcessingResult(success=True, data=clean_data)
    
    except Exception as e:
        return ProcessingResult(success=False, error="Processing failed")

# Test cases
test_cases = [
    {"user_id": 1, "email": "Alice@Example.com", "age": 25},  # Valid
    {"user_id": 2, "email": "bob@test.com"},                   # Missing age (ok)
    {"user_id": 3, "age": 30},                                 # Missing email
    {"email": "charlie@test.com", "age": 20},                  # Missing user_id
]

print("Processing results:")
for i, data in enumerate(test_cases):
    result = process_user_data(data)
    print(f"  Case {i+1}: success={result.success}, error={result.error}")
```

---

**Task 2a: Identify the Flaw**

Run the code. Notice that Case 1 (which has all valid data) reports `success=False`.

1. Why does the valid test case fail?
2. Which line contains the bug?
3. Why doesn't this bug cause a clear error message?

---

**Task 2b: Explain the Mechanism**

1. What exception is raised when the code tries to access `raw_data["emal"]`?
2. Why does `except Exception` catch this?
3. Why is the error message "Processing failed" useless for debugging?

---

**Task 2c: Instrument to Prove Diagnosis**

Modify the except block to reveal what's actually going wrong:

```python
except Exception as e:
    # Add instrumentation
    print(f"DEBUG: Exception type: {type(e).__name__}")
    print(f"DEBUG: Exception message: {e}")
    return ProcessingResult(success=False, error="Processing failed")
```

Show the output that reveals the hidden bug.

---

**Task 2d: Fix the Code**

Fix the code with proper exception handling:

1. Fix the typo
2. Use specific exception handling:
   - Missing required fields → specific error message
   - Validation failures → specific error message  
   - Let programming bugs crash

---

**Task 2e: Write Verification Test**

Write tests that verify correct error handling:

```python
def test_missing_email_gives_specific_error():
    """Should report 'Missing required field: email', not generic error."""
    result = process_user_data({"user_id": 1, "age": 25})
    assert result.success == False
    assert "email" in result.error.lower()  # Error message mentions the field

def test_valid_data_succeeds():
    """Valid data should process successfully."""
    result = process_user_data({
        "user_id": 1, 
        "email": "test@example.com", 
        "age": 25
    })
    assert result.success == True
    assert result.data["id"] == 1
```

---

# Level 2 Solutions

## Solution 2.1: The Decorator That Forgot


**Task 2a: Identify the Flaw**
1.  **What is wrong?**
    *   **Function Name:** Instead of `fetch_user`, the name is now `wrapper`.
    *   **Docstring:** The docstring is missing (or would show `wrapper`'s docstring if it had one).
    *   **Help:** `help(fetch_user)` shows the signature `(*args: Any, **kwargs: Any) -> Any`, losing the specific parameter information (`user_id`).

2.  **Why is this problematic?**
    *   **Debugging:** If an error occurs, the traceback will say the error happened in `wrapper`, making it hard to know *which* decorated function actually failed.
    *   **Documentation:** Tools that auto-generate documentation (like Sphinx) will fail to read the descriptions of your API endpoints.


**Task 2b: Explain the Mechanism:**

1.  **Object Identity:** After decoration, the variable `fetch_user` no longer points to your original function. It points to the `wrapper` closure created inside `timed`.
2.  **Missing Metadata:** The original function (`func`) is stored safely inside the wrapper's closure, but Python does not automatically copy attributes like `__name__` or `__doc__` from `func` to `wrapper`.
3.  **Useless Help:** Since `fetch_user` is now `wrapper`, the `help()` function inspects `wrapper`. Since `wrapper` accepts `*args, **kwargs` to be generic, that is exactly what `help()` reports.

**Task 2c: Instrument to Prove Diagnosis:**
```python
# Add instrumentation to prove the identity problem
print(f"fetch_user is wrapper: {fetch_user.__name__ == 'wrapper'}")
print(f"Original function name preserved: {fetch_user.__name__ == 'fetch_user'}")

# Output:
# fetch_user is wrapper: True
# Original function name preserved: False
```

**Task 2d: Fix the Code:**
We use `functools.wraps`. This is a helper decorator specifically designed to copy metadata (name, docstring, annotations) from the original function to the wrapper.

```python
import time
import functools  # <--- 1. Import this
from typing import Callable, Any

def timed(func: Callable) -> Callable:
    """Decorator that logs execution time"""
    
    @functools.wraps(func)  # <--- 2. Apply this to the wrapper
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - start
        print(f"{func.__name__} took {elapsed:.3f}s")
        return result
    
    return wrapper

@timed
def fetch_user(user_id: int) -> dict:
    """Fetch user from database by ID."""
    time.sleep(0.1)
    return {"id": user_id, "name": "Alice"}
```

**Task 2e: Write Verification Test:**
```python
def test_metadata_preserved():
    """
    This test verifies that metadata is copied correctly.
    """
    @timed
    def sample_function():
        """Sample docstring."""
        pass
    
    # Assertions
    assert sample_function.__name__ == "sample_function", "Name was lost!"
    assert sample_function.__doc__ == "Sample docstring.", "Docstring was lost!"
    print("Test Passed: Metadata is preserved.")

# Run the test
test_metadata_preserved()
```

---

## Solution 2.2: The Exception That Hides Everything

**Task 2a: Identify the Flaw:**

1.  **Why does the valid case fail?** 
- The valid case fails because of a typo in the code: `raw_data["emal"]` (missing 'i'). This raises a `KeyError`.
2.  **Which line?** 
- `if "@" not in raw_data["emal"]:`
3.  **Why no clear error?** 
- The code is wrapped in `try... except Exception`. This catches the `KeyError` caused by the typo, assumes it was a logic error, and returns the generic "Processing failed" message defined in the `except` block.

**Task 2b: Explain the Mechanism:**

1.  **Exception Raised:** 
- `KeyError: 'emal'` (because the key 'emal' does not exist in the dictionary).
2.  **Why it's caught:** 
- `Exception` is the base class for almost all errors in Python (including `KeyError`, `ValueError`, `TypeError`, etc.). The `except` block indiscriminately catches them all.
3.  **Why useless:** 
- The message "Processing failed" discards the stack trace. You don't know if the failure was due to bad data, a database connection drop, or a typo in the variable name.

**Task 2c: Instrument to Prove Diagnosis:**
```python
    except Exception as e:
        # Add instrumentation
        print(f"DEBUG: Exception type: {type(e).__name__}")
        print(f"DEBUG: Exception message: {e}")
        return ProcessingResult(success=False, error="Processing failed")

# Output for Case 1:
# DEBUG: Exception type: KeyError
# DEBUG: Exception message: 'emal'
```


**Task 2d: Fix the Code:**
The fix involves two steps:
1.  Fix the typo.
2.  **Crucially:** Remove the broad `except Exception` so that future typos crash loudly, and handle expected data errors explicitly.

```python
def process_user_data(raw_data: dict) -> ProcessingResult:
    """Process raw user data into clean format."""
    
    # 1. Handle Missing Data Explicitly
    # We check for required keys first. If they are missing, it's a data error.
    required_fields = ["user_id", "email"]
    for field in required_fields:
        if field not in raw_data:
            return ProcessingResult(success=False, error=f"Missing field: {field}")

    # 2. Extract Data (Safe now because we checked above)
    user_id = raw_data["user_id"]
    email = raw_data["email"]
    
    # 3. Validate Logic
    # Fix the typo here: "email" instead of "emal"
    if "@" not in email:
        return ProcessingResult(success=False, error="Invalid email")
    
    age = raw_data.get("age", 0)
    if age < 0:
        return ProcessingResult(success=False, error="Invalid age")
    
    # 4. Build Result
    # Note: We removed the try/except block entirely. 
    # If I make a typo now, the program will CRASH, which is exactly what I want
    # so I can fix the bug immediately.
    clean_data = {
        "id": user_id,
        "email": email.lower().strip(),
        "age": age,
        "is_adult": age >= 18
    }
    
    return ProcessingResult(success=True, data=clean_data)
```

**Task 2e: Write Verification Test:**
```python
def test_missing_email_gives_specific_error():
    """Should report 'Missing field: email', not generic error."""
    # Missing email
    result = process_user_data({"user_id": 1, "age": 25})
    
    assert result.success is False
    assert "email" in result.error.lower()
    print("Test Passed: Missing field handled correctly.")

def test_valid_data_succeeds():
    """Valid data should process successfully."""
    result = process_user_data({
        "user_id": 1, 
        "email": "test@example.com", 
        "age": 25
    })
    
    assert result.success is True
    assert result.data["id"] == 1
    print("Test Passed: Valid data succeeds.")

# Run tests
test_missing_email_gives_specific_error()
test_valid_data_succeeds()
```

---

# LEVEL 2.5: EXTEND EXISTING CODE

## Given Code

**Context:** Your teammate wrote this database connection context manager. It's been working in production for 2 months. The tests pass. Your task is to add features WITHOUT breaking existing behavior.

```python
from contextlib import contextmanager
from dataclasses import dataclass, field
from typing import Generator, Optional
from datetime import datetime
import time

@dataclass
class QueryResult:
    """Result of a database query."""
    data: list[dict]
    rows_affected: int
    execution_time: float

class DatabaseError(Exception):
    """Base exception for database errors."""
    pass

class ConnectionError(DatabaseError):
    """Failed to connect to database."""
    pass

class QueryError(DatabaseError):
    """Query execution failed."""
    pass

class Database:
    """Simulated database connection."""
    
    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self.connected = False
        self._query_count = 0
    
    def connect(self) -> None:
        """Establish database connection."""
        if self.connected:
            raise ConnectionError("Already connected")
        print(f"[DB] Connecting to {self.host}:{self.port}")
        time.sleep(0.1)  # Simulate connection time
        self.connected = True
    
    def disconnect(self) -> None:
        """Close database connection."""
        if not self.connected:
            return  # Idempotent - safe to call multiple times
        print(f"[DB] Disconnecting from {self.host}:{self.port}")
        self.connected = False
    
    def execute(self, query: str) -> QueryResult:
        """Execute a SQL query."""
        if not self.connected:
            raise ConnectionError("Not connected to database")
        
        start = time.time()
        self._query_count += 1
        
        # Simulate query execution
        time.sleep(0.05)
        
        # Simulate failure for certain queries
        if "INVALID" in query:
            raise QueryError(f"Syntax error in query: {query}")
        
        elapsed = time.time() - start
        
        # Return simulated results
        return QueryResult(
            data=[{"id": 1, "name": "test"}],
            rows_affected=1,
            execution_time=elapsed
        )
    
    @property
    def query_count(self) -> int:
        return self._query_count


@contextmanager
def database_connection(host: str, port: int) -> Generator[Database, None, None]:
    """
    Context manager for database connections.
    
    Ensures the connection is properly closed even if an error occurs.
    
    Usage:
        with database_connection("localhost", 5432) as db:
            result = db.execute("SELECT * FROM users")
    """
    db = Database(host, port)
    db.connect()
    try:
        yield db
    finally:
        db.disconnect()


# ============== EXISTING TESTS (must continue to pass) ==============

def test_basic_connection():
    """Connection opens and closes correctly."""
    with database_connection("localhost", 5432) as db:
        assert db.connected == True
        result = db.execute("SELECT 1")
        assert result.rows_affected == 1
    assert db.connected == False

def test_cleanup_on_error():
    """Connection closes even when query fails."""
    db_ref = None
    try:
        with database_connection("localhost", 5432) as db:
            db_ref = db
            db.execute("INVALID QUERY")
    except QueryError:
        pass  # Expected
    
    assert db_ref is not None
    assert db_ref.connected == False  # Must be disconnected despite error

def test_query_execution():
    """Basic query returns results."""
    with database_connection("localhost", 5432) as db:
        result = db.execute("SELECT * FROM users")
        assert len(result.data) > 0
        assert result.execution_time > 0

# Run existing tests
if __name__ == "__main__":
    test_basic_connection()
    print("✅ test_basic_connection passed")
    
    test_cleanup_on_error()
    print("✅ test_cleanup_on_error passed")
    
    test_query_execution()
    print("✅ test_query_execution passed")
    
    print("\n✅ All existing tests pass!")
```

---

## Extension A: Connection Retry

**Requirement:** The database might be temporarily unavailable. Add automatic retry logic to `database_connection()` that attempts to connect up to 3 times with a 0.1 second delay between attempts.

**Constraint:** Only retry on initial connection failure. Do NOT retry queries.

**To simulate connection failure**, modify your Database class:
```python
# Add this to simulate flaky connections
_connection_attempts = 0

def connect(self) -> None:
    global _connection_attempts
    _connection_attempts += 1
    
    # Fail first 2 attempts, succeed on 3rd
    if _connection_attempts < 3:
        raise ConnectionError(f"Connection refused (attempt {_connection_attempts})")
    
    # ... rest of original connect code ...
```

**New test your code must pass:**
```python
def test_connection_retry():
    """Should retry failed connections up to 3 times."""
    global _connection_attempts
    _connection_attempts = 0
    
    # Should succeed on 3rd attempt
    with database_connection("localhost", 5432) as db:
        assert db.connected == True
    
    assert _connection_attempts == 3  # Took 3 attempts

def test_connection_gives_up_after_max_retries():
    """Should raise after max retries exceeded."""
    # Simulate permanent failure by making threshold higher
    global _connection_attempts  
    _connection_attempts = 0
    
    # Modify to always fail
    try:
        with database_connection("localhost", 5432, max_retries=2) as db:
            pass
        assert False, "Should have raised"
    except ConnectionError as e:
        assert "attempts" in str(e).lower() or "failed" in str(e).lower()
```

**All existing tests must still pass.**

---

## Extension B: Query Timing Log

**Requirement:** Add optional query logging that records all executed queries with their execution times. The log should be accessible after the context manager exits.

**Constraint:** Logging must be opt-in (disabled by default) to avoid overhead.

**New test your code must pass:**
```python
def test_query_logging():
    """Should log queries when enabled."""
    with database_connection("localhost", 5432, log_queries=True) as db:
        db.execute("SELECT * FROM users")
        db.execute("SELECT * FROM orders")
        query_log = db.get_query_log()
    
    assert len(query_log) == 2
    assert query_log[0]["query"] == "SELECT * FROM users"
    assert query_log[0]["execution_time"] > 0
    assert query_log[1]["query"] == "SELECT * FROM orders"

def test_logging_disabled_by_default():
    """Query logging should be off by default."""
    with database_connection("localhost", 5432) as db:
        db.execute("SELECT * FROM users")
        # Should return empty or raise AttributeError
        log = db.get_query_log() if hasattr(db, 'get_query_log') else []
    
    assert len(log) == 0
```

**All previous tests must still pass.**

---

## Extension C: Transaction Support

**Requirement:** Add transaction support with automatic rollback on error.

**Constraint:** 
- `commit()` should be called if block completes successfully
- `rollback()` should be called if an exception occurs
- Must work correctly with query logging from Extension B

**New test your code must pass:**
```python
def test_transaction_commits_on_success():
    """Transaction commits when block succeeds."""
    with database_connection("localhost", 5432) as db:
        with db.transaction():
            db.execute("INSERT INTO users VALUES (1, 'test')")
        assert db.last_transaction_status == "committed"

def test_transaction_rollback_on_error():
    """Transaction rolls back when block fails."""
    with database_connection("localhost", 5432) as db:
        try:
            with db.transaction():
                db.execute("INSERT INTO users VALUES (1, 'test')")
                raise ValueError("Application error")
        except ValueError:
            pass
        assert db.last_transaction_status == "rolled_back"

def test_nested_usage_with_logging():
    """Transaction works with query logging enabled."""
    with database_connection("localhost", 5432, log_queries=True) as db:
        with db.transaction():
            db.execute("SELECT * FROM users")
        
        log = db.get_query_log()
        assert len(log) >= 1  # At least the SELECT query
```

**All previous tests must still pass.**

---

## Regression Contract

After completing all extensions, the following must be true:

1. All original tests pass unchanged
2. All new extension tests pass
3. Original functionality works identically:
   - Basic connection/disconnection
   - Query execution
   - Cleanup on error
4. Extensions are backward compatible (new parameters have defaults)

---

# Level 2.5 Solutions

## Extension A Solution: Connection Retry

```python
import time
from contextlib import contextmanager
from typing import Generator

# Track connection attempts for testing
_connection_attempts = 0

class Database:
    # ... existing code ...
    
    def connect(self) -> None:
        """Establish database connection."""
        global _connection_attempts
        _connection_attempts += 1
        
        if self.connected:
            raise ConnectionError("Already connected")
        
        # Simulate flaky connection (fail first 2 attempts)
        if _connection_attempts < 3:
            raise ConnectionError(f"Connection refused (attempt {_connection_attempts})")
        
        print(f"[DB] Connecting to {self.host}:{self.port}")
        time.sleep(0.1)
        self.connected = True


@contextmanager
def database_connection(
    host: str, 
    port: int, 
    max_retries: int = 3,
    retry_delay: float = 0.1
) -> Generator[Database, None, None]:
    """
    Context manager for database connections with retry logic.
    """
    db = Database(host, port)
    
    last_error = None
    for attempt in range(1, max_retries + 1):
        try:
            db.connect()
            break  # Success!
        except ConnectionError as e:
            last_error = e
            if attempt < max_retries:
                print(f"[DB] Connection attempt {attempt} failed, retrying...")
                time.sleep(retry_delay)
    else:
        # Loop completed without break = all retries failed
        raise ConnectionError(
            f"Failed to connect after {max_retries} attempts: {last_error}"
        )
    
    try:
        yield db
    finally:
        db.disconnect()
```

---

## Extension B Solution: Query Logging

```python
from dataclasses import dataclass, field
from typing import Optional
from datetime import datetime

@dataclass
class Database:
    host: str
    port: int
    connected: bool = False
    _query_count: int = 0
    _log_queries: bool = False
    _query_log: list[dict] = field(default_factory=list)
    
    def enable_query_logging(self) -> None:
        self._log_queries = True
        self._query_log = []
    
    def get_query_log(self) -> list[dict]:
        return self._query_log.copy() if self._log_queries else []
    
    def execute(self, query: str) -> QueryResult:
        if not self.connected:
            raise ConnectionError("Not connected to database")
        
        start = time.time()
        self._query_count += 1
        
        time.sleep(0.05)  # Simulate execution
        
        if "INVALID" in query:
            raise QueryError(f"Syntax error in query: {query}")
        
        elapsed = time.time() - start
        
        # Log if enabled
        if self._log_queries:
            self._query_log.append({
                "query": query,
                "execution_time": elapsed,
                "timestamp": datetime.now().isoformat()
            })
        
        return QueryResult(
            data=[{"id": 1, "name": "test"}],
            rows_affected=1,
            execution_time=elapsed
        )


@contextmanager
def database_connection(
    host: str,
    port: int,
    max_retries: int = 3,
    retry_delay: float = 0.1,
    log_queries: bool = False  # New parameter
) -> Generator[Database, None, None]:
    db = Database(host=host, port=port)
    
    if log_queries:
        db.enable_query_logging()
    
    # ... retry logic from Extension A ...
    
    try:
        yield db
    finally:
        db.disconnect()
```

---

## Extension C Solution: Transaction Support

```python
from contextlib import contextmanager

class Database:
    # ... previous code ...
    
    last_transaction_status: Optional[str] = None
    _in_transaction: bool = False
    
    @contextmanager
    def transaction(self) -> Generator[None, None, None]:
        """Context manager for database transactions."""
        if self._in_transaction:
            raise DatabaseError("Nested transactions not supported")
        
        self._in_transaction = True
        self._begin_transaction()
        
        try:
            yield
            self._commit_transaction()
            self.last_transaction_status = "committed"
        except Exception:
            self._rollback_transaction()
            self.last_transaction_status = "rolled_back"
            raise
        finally:
            self._in_transaction = False
    
    def _begin_transaction(self) -> None:
        print("[DB] BEGIN TRANSACTION")
    
    def _commit_transaction(self) -> None:
        print("[DB] COMMIT")
    
    def _rollback_transaction(self) -> None:
        print("[DB] ROLLBACK")
```

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## Scenario: API Response Builder

You're building a response builder for a REST API. The system needs to handle various data sources and transform them into consistent API responses. You have strict requirements around error handling and data integrity.

**Input:** Raw data from various sources (database, cache, external API)
**Output:** Standardized API response with proper error handling

**Constraints:**

| Constraint | Specification | Why It Matters |
|------------|---------------|----------------|
| Type Safety | All response fields must have correct types | Frontend relies on schema |
| Error Clarity | Each error type needs specific, actionable message | Support team needs to diagnose |
| Immutability | Response objects cannot be modified after creation | Prevents bugs in middleware |
| Extensibility | New response types must be addable without modifying core | Open/closed principle |

---

## Starter Code

```python
from dataclasses import dataclass, field
from typing import Optional, Any, TypeVar, Generic
from datetime import datetime
from enum import Enum, auto
from abc import ABC, abstractmethod

# ============== TYPE DEFINITIONS ==============

class StatusCode(Enum):
    """HTTP-like status codes for API responses."""
    OK = 200
    CREATED = 201
    BAD_REQUEST = 400
    UNAUTHORIZED = 401
    NOT_FOUND = 404
    INTERNAL_ERROR = 500

T = TypeVar('T')

# ============== EXCEPTION HIERARCHY (provided) ==============

class APIError(Exception):
    """Base exception for all API errors."""
    status_code: StatusCode = StatusCode.INTERNAL_ERROR
    
    def __init__(self, message: str, details: Optional[dict] = None):
        self.message = message
        self.details = details or {}
        super().__init__(message)

class ValidationError(APIError):
    """Request validation failed."""
    status_code = StatusCode.BAD_REQUEST

class NotFoundError(APIError):
    """Requested resource not found."""
    status_code = StatusCode.NOT_FOUND

class AuthenticationError(APIError):
    """Authentication failed."""
    status_code = StatusCode.UNAUTHORIZED

# ============== YOUR TASK: Implement these ==============

# TODO: Implement a frozen dataclass for successful responses
# Requirements:
# - Generic over data type T
# - Fields: success (bool), status_code (StatusCode), data (T), timestamp (datetime)
# - Must be immutable (frozen)
# - timestamp should default to current time

# TODO: Implement a frozen dataclass for error responses
# Requirements:
# - Fields: success (bool), status_code (StatusCode), error_type (str), 
#           message (str), details (dict), timestamp (datetime)
# - Must be immutable (frozen)

# TODO: Implement a response builder context manager
# Requirements:
# - Catches exceptions and converts to appropriate error responses
# - Returns success response if no exception
# - Logs all responses (print for now)

# ============== TEST HARNESS (do not modify) ==============

def simulate_database_fetch(user_id: int) -> dict:
    """Simulate fetching user from database."""
    if user_id == 404:
        raise NotFoundError(f"User {user_id} not found", {"user_id": user_id})
    if user_id == 401:
        raise AuthenticationError("Invalid API key")
    if user_id < 0:
        raise ValidationError("User ID must be positive", {"field": "user_id", "value": user_id})
    
    return {"id": user_id, "name": "Test User", "email": "test@example.com"}


def main():
    print("=== Test 1: Successful response ===")
    # Should return SuccessResponse with user data
    
    print("\n=== Test 2: Not found error ===")
    # Should return ErrorResponse with NOT_FOUND status
    
    print("\n=== Test 3: Validation error ===")  
    # Should return ErrorResponse with BAD_REQUEST status
    
    print("\n=== Test 4: Authentication error ===")
    # Should return ErrorResponse with UNAUTHORIZED status
    
    print("\n=== Test 5: Unexpected error ===")
    # Non-APIError exceptions should become INTERNAL_ERROR
    
    # Verify immutability
    print("\n=== Test 6: Immutability check ===")
    # Attempting to modify a response should raise FrozenInstanceError

if __name__ == "__main__":
    main()
```

---

## Phase 3a: Core Data Classes

Implement `SuccessResponse` and `ErrorResponse` as frozen dataclasses.

**Requirements:**
- Both must be immutable (frozen=True)
- SuccessResponse should be generic over the data type
- Both need automatic timestamp defaulting to now
- ErrorResponse should be constructable from an APIError exception

**Verification:** Run the immutability test - modifying any field should raise an error.

---

## Phase 3b: Response Builder Context Manager

Implement a context manager that:
1. Yields a "response collector" 
2. If the block succeeds, creates a SuccessResponse
3. If an APIError is raised, creates an appropriate ErrorResponse
4. If any other exception is raised, creates an INTERNAL_ERROR response
5. Logs all responses (print for now)

```python
@contextmanager
def build_response() -> Generator[???]:
    """
    Context manager for building API responses.
    
    Usage:
        with build_response() as response:
            data = fetch_something()
            response.set_data(data)
        
        # response.result contains SuccessResponse or ErrorResponse
    """
    # Your implementation
```

**Verification:** All test cases produce appropriate response types.

---

## Phase 3c: Exception-to-Response Decorator

Create a decorator that can be applied to endpoint functions:

```python
@api_endpoint
def get_user(user_id: int) -> dict:
    if user_id < 0:
        raise ValidationError("User ID must be positive")
    return {"id": user_id, "name": "Test"}

# Calling get_user(123) returns SuccessResponse
# Calling get_user(-1) returns ErrorResponse (not raising)
```

**Requirements:**
- Must preserve function metadata (@wraps)
- Must work with the exception hierarchy
- Return type should be Union[SuccessResponse, ErrorResponse]

---

## Phase 3d: Write Your Own Edge Case Test

Write ONE test that verifies an edge case not covered by the provided tests:

```python
def test_my_edge_case():
    """
    Edge case: [Description]
    Why it matters: [Explanation]
    """
    # Your test
```

Examples of good edge cases:
- None returned from function (is this success or error?)
- Exception with None message
- Nested exceptions (exception in error handler)
- Empty details dict vs missing details

---

# Level 3 Solution

## Phase 3a Solution: Data Classes

```python
from dataclasses import dataclass, field
from typing import Generic, TypeVar, Optional, Union
from datetime import datetime

T = TypeVar('T')

@dataclass(frozen=True)
class SuccessResponse(Generic[T]):
    """Immutable response for successful API calls."""
    data: T
    success: bool = field(default=True, init=False)
    status_code: StatusCode = StatusCode.OK
    timestamp: datetime = field(default_factory=datetime.now)
    
    def __post_init__(self):
        # For frozen dataclasses, use object.__setattr__
        object.__setattr__(self, 'success', True)


@dataclass(frozen=True)
class ErrorResponse:
    """Immutable response for failed API calls."""
    message: str
    error_type: str
    status_code: StatusCode = StatusCode.INTERNAL_ERROR
    details: dict = field(default_factory=dict)
    success: bool = field(default=False, init=False)
    timestamp: datetime = field(default_factory=datetime.now)
    
    @classmethod
    def from_exception(cls, exc: Exception) -> "ErrorResponse":
        """Create ErrorResponse from an exception."""
        if isinstance(exc, APIError):
            return cls(
                message=exc.message,
                error_type=type(exc).__name__,
                status_code=exc.status_code,
                details=exc.details
            )
        else:
            return cls(
                message=str(exc) or "An unexpected error occurred",
                error_type=type(exc).__name__,
                status_code=StatusCode.INTERNAL_ERROR,
                details={}
            )
```

## Phase 3b Solution: Context Manager

```python
from contextlib import contextmanager
from typing import Generator

@dataclass
class ResponseCollector:
    """Collects data and produces final response."""
    _data: Optional[Any] = None
    _result: Optional[Union[SuccessResponse, ErrorResponse]] = None
    
    def set_data(self, data: Any) -> None:
        self._data = data
    
    @property
    def result(self) -> Union[SuccessResponse, ErrorResponse]:
        if self._result is None:
            raise RuntimeError("Response not yet built")
        return self._result


@contextmanager
def build_response() -> Generator[ResponseCollector, None, None]:
    """
    Context manager for building API responses.
    
    Catches exceptions and converts to appropriate responses.
    """
    collector = ResponseCollector()
    
    try:
        yield collector
        # Block completed successfully
        collector._result = SuccessResponse(data=collector._data)
        print(f"[Response] Success: {collector._result}")
        
    except APIError as e:
        collector._result = ErrorResponse.from_exception(e)
        print(f"[Response] API Error: {collector._result}")
        
    except Exception as e:
        collector._result = ErrorResponse.from_exception(e)
        print(f"[Response] Unexpected Error: {collector._result}")


# Usage example
with build_response() as response:
    data = simulate_database_fetch(123)
    response.set_data(data)

print(f"Final result: {response.result}")
```

## Phase 3c Solution: Decorator

```python
from functools import wraps
from typing import Callable, Union

def api_endpoint(func: Callable[..., T]) -> Callable[..., Union[SuccessResponse[T], ErrorResponse]]:
    """
    Decorator that wraps function results in API responses.
    
    Successful returns become SuccessResponse.
    APIErrors become appropriate ErrorResponse.
    Other exceptions become INTERNAL_ERROR response.
    """
    @wraps(func)
    def wrapper(*args, **kwargs) -> Union[SuccessResponse[T], ErrorResponse]:
        try:
            result = func(*args, **kwargs)
            return SuccessResponse(data=result)
        except APIError as e:
            return ErrorResponse.from_exception(e)
        except Exception as e:
            return ErrorResponse.from_exception(e)
    
    return wrapper


# Usage
@api_endpoint
def get_user(user_id: int) -> dict:
    """Get user by ID."""
    return simulate_database_fetch(user_id)


# Tests
result1 = get_user(123)
assert isinstance(result1, SuccessResponse)
assert result1.data["id"] == 123

result2 = get_user(404)
assert isinstance(result2, ErrorResponse)
assert result2.status_code == StatusCode.NOT_FOUND

result3 = get_user(-1)
assert isinstance(result3, ErrorResponse)
assert result3.status_code == StatusCode.BAD_REQUEST

print("✅ All api_endpoint tests pass!")
```

## Phase 3d: Example Edge Case Tests

```python
def test_none_return_value():
    """
    Edge case: Function returns None explicitly
    Why it matters: None is falsy, but returning None should be valid success
    """
    @api_endpoint
    def return_none() -> None:
        return None
    
    result = return_none()
    assert isinstance(result, SuccessResponse)
    assert result.data is None
    assert result.success == True


def test_empty_error_message():
    """
    Edge case: Exception with empty message
    Why it matters: str(exc) returns "" which could cause display issues
    """
    @api_endpoint
    def raise_empty_error():
        raise ValueError("")  # Empty message
    
    result = raise_empty_error()
    assert isinstance(result, ErrorResponse)
    assert result.message != ""  # Should have fallback message


def test_response_immutability():
    """
    Edge case: Attempting to modify response after creation
    Why it matters: Mutable responses could be corrupted by middleware
    """
    response = SuccessResponse(data={"count": 1})
    
    try:
        response.data = {"count": 2}
        assert False, "Should have raised FrozenInstanceError"
    except AttributeError:
        pass  # Expected for frozen dataclass
```

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## Exercise 4: When NOT to Use These Patterns

### Part A: The Dataclass Trap

A developer heard "dataclasses reduce boilerplate" and wrote this:

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class MathCalculator:
    """Performs mathematical calculations."""
    last_result: Optional[float] = None
    operation_count: int = 0
    
    def add(self, a: float, b: float) -> float:
        self.operation_count += 1
        self.last_result = a + b
        return self.last_result
    
    def multiply(self, a: float, b: float) -> float:
        self.operation_count += 1
        self.last_result = a * b
        return self.last_result
    
    def get_stats(self) -> dict:
        return {
            "operations": self.operation_count,
            "last_result": self.last_result
        }


calc = MathCalculator()
print(calc.add(2, 3))      # 5
print(calc.multiply(4, 5)) # 20
print(calc.get_stats())    # {'operations': 2, 'last_result': 20}
print(calc)                # MathCalculator(last_result=20, operation_count=2)
```

**Questions:**

1. **What does @dataclass provide here?** List specifically what the decorator generates.

2. **What do we NOT need?** Of the generated methods, which are actually useful for this class?

3. **What's wrong with this design?** Is `MathCalculator` really a "data class"?

4. **Rewrite without @dataclass.** How many lines does it actually save?

---

### Part B: The Decorator Trap

A developer heard "decorators add behavior cleanly" and wrote this:

```python
from functools import wraps
from typing import Callable

def add_one(func: Callable[[int], int]) -> Callable[[int], int]:
    """Add 1 to the function's return value."""
    @wraps(func)
    def wrapper(x: int) -> int:
        return func(x) + 1
    return wrapper

def multiply_by_two(func: Callable[[int], int]) -> Callable[[int], int]:
    """Multiply the function's return value by 2."""
    @wraps(func)
    def wrapper(x: int) -> int:
        return func(x) * 2
    return wrapper

@add_one
@multiply_by_two
def square(x: int) -> int:
    """Return x squared."""
    return x * x

# What does square(3) return?
result = square(3)
print(f"square(3) = {result}")

# What about square(4)?
result2 = square(4)
print(f"square(4) = {result2}")
```

**Questions:**

1. **Predict:** What does `square(3)` return? Walk through the execution.

2. **The problem:** How hard is it to understand what `square()` does now? What if there were 5 decorators?

3. **When is this pattern actually harmful?** List scenarios where stacking decorators obscures rather than clarifies.

4. **Better alternative:** How would you achieve this same transformation without decorators?

---

### Part C: Create a Decision Rule

Based on your analysis, complete this framework:

**Use dataclasses when:**
- [ ] The class is primarily a container for ______
- [ ] You need automatic ______, ______, or ______ methods
- [ ] The data is ______ (rarely modified after creation)

**Avoid dataclasses when:**
- [ ] The class is primarily about ______ not data
- [ ] You have complex ______ logic
- [ ] You don't need the generated ______

**Use decorators when:**
- [ ] The behavior is ______ (applies to many functions)
- [ ] The modification is ______ (same transform every time)
- [ ] The decorator name clearly describes ______

**Avoid decorators when:**
- [ ] The logic is ______ to the function (only used once)
- [ ] Stacking multiple decorators makes code ______
- [ ] The transformation could be done with a simple ______

---

# Level 4 Solutions

## Part A Solution

1. **What @dataclass provides:** `__init__`, `__repr__`, `__eq__` (and optionally `__hash__`, `__lt__`, etc.)

2. **What we don't need:**
   - `__eq__`: We'd never compare two calculators for equality
   - `__repr__`: Useful for debugging, but not essential
   - The auto-generated `__init__` saves maybe 4 lines

3. **What's wrong:** `MathCalculator` is NOT a data class. It's a service/utility class that happens to have some state. The `@dataclass` decorator implies "this class represents data," which is misleading.

4. **Without @dataclass:**
```python
class MathCalculator:
    def __init__(self):
        self.last_result: Optional[float] = None
        self.operation_count: int = 0
    
    # ... rest identical ...
```
Savings: ~3 lines. The decorator added cognitive overhead for minimal benefit.

## Part B Solution

1. **Execution trace:**
   ```
   square(3)
   → @add_one wraps @multiply_by_two(square)
   → add_one.wrapper(3)
   → multiply_by_two.wrapper(3) + 1
   → (square(3) * 2) + 1
   → (9 * 2) + 1
   → 19
   ```

2. **The problem:** To understand `square(3)`, you must mentally simulate THREE nested function calls. With 5 decorators, it becomes nearly impossible.

3. **When harmful:**
   - Data transformation decorators (obscure what data looks like)
   - Decorators with side effects (order matters but isn't obvious)
   - Business logic in decorators (should be explicit)

4. **Better alternative:**
```python
def square(x: int) -> int:
    return x * x

def transformed_square(x: int) -> int:
    """Square, multiply by 2, add 1."""
    return (square(x) * 2) + 1
```
Now the transformation is **explicit** and **documented**.

## Part C: Decision Framework

**Use dataclasses when:**
- [x] The class is primarily a container for **related data fields**
- [x] You need automatic **`__init__`**, **`__repr__`**, or **`__eq__`** methods
- [x] The data is **stable** (rarely modified after creation)

**Avoid dataclasses when:**
- [x] The class is primarily about **behavior/methods** not data
- [x] You have complex **initialization** logic
- [x] You don't need the generated **comparison/representation methods**

**Use decorators when:**
- [x] The behavior is **cross-cutting** (applies to many functions)
- [x] The modification is **consistent** (same transform every time)
- [x] The decorator name clearly describes **what it does**

**Avoid decorators when:**
- [x] The logic is **specific** to the function (only used once)
- [x] Stacking multiple decorators makes code **hard to trace**
- [x] The transformation could be done with a simple **function call**

---

# LEVEL 5: THE TEACH-BACK

## Scenario: Code Review

You're reviewing a pull request from a junior developer. They've implemented a "user service" with error handling:

```python
from dataclasses import dataclass

# Junior's code with comments explaining their thinking

@dataclass
class UserService:
    """Service for user operations.
    
    I used @dataclass because it automatically creates __init__ for me
    and makes the code more 'Pythonic'. The db connection is stored
    as an attribute so we can use it in all methods.
    """
    db_connection: object  # The database connection
    cache: dict = {}  # Cache for user lookups - I initialize it here so all instances share it for efficiency
    
    def get_user(self, user_id: int):
        """
        Get user by ID. I use a broad except here to make sure 
        the service never crashes - it's better to return None
        than to crash the whole application.
        """
        try:
            # Check cache first for speed
            if user_id in self.cache:
                return self.cache[user_id]
            
            # Query database
            user = self.db_connection.query(f"SELECT * FROM users WHERE id = {user_id}")
            
            # Cache for next time
            self.cache[user_id] = user
            return user
        except:
            # Never crash! Return None if anything goes wrong
            return None
    
    def create_user(self, name: str, email: str):
        """
        Create a new user. I don't need @wraps on this because
        it's not a decorator, it's just a regular method.
        """
        def validate_email(email):
            """Quick validation."""
            return "@" in email
        
        if not validate_email(email):
            return None  # Invalid email, return None instead of raising
        
        try:
            self.db_connection.insert(f"INSERT INTO users (name, email) VALUES ('{name}', '{email}')")
            return True
        except:
            return None

# Usage
service = UserService(db_connection=get_db())
user = service.get_user(123)
if user is None:
    print("User not found or error occurred")  # Can't tell which!
```

---

## Your Task: Write a Code Review Response

Your response must:

1. **Correct each misconception** specifically:
   - Quote their comment, then explain the correct understanding

2. **Identify security issues** (there are at least 2 critical ones)

3. **Explain the problems** with their error handling approach

4. **Provide corrected code** that demonstrates:
   - Proper use (or non-use) of dataclass
   - Custom exceptions with context
   - Specific exception handling
   - Safe database queries

5. **Give them a mental model** for when to use each pattern

---

# Level 5 Model Response

## Code Review: User Service

Hey! Thanks for this PR. I can see you're thinking about code organization and error handling, which is great. I have some feedback that should help make this more robust.

---

## Misconception Corrections

**1. "I used @dataclass because it automatically creates __init__"**

@dataclass is designed for classes that are primarily **data containers**—like a User, Config, or Response. Your UserService is a **behavior class** (it does things, not holds data). Using @dataclass here is like using a screwdriver to hammer a nail—it might work, but it's the wrong tool.

Better approach: Use a regular class with explicit `__init__`:
```python
class UserService:
    def __init__(self, db_connection: DatabaseConnection):
        self.db_connection = db_connection
        self._cache: dict[int, User] = {}  # Instance-specific cache
```

**2. "cache: dict = {} ... so all instances share it for efficiency"**

This is the mutable default trap! With `cache: dict = {}`, ALL UserService instances share the SAME dictionary. If you create two services for different databases, they'll corrupt each other's caches.

```python
service1 = UserService(db1)
service2 = UserService(db2)
service1.cache[1] = user_from_db1
# Now service2.cache[1] also contains user_from_db1! Bug!
```

**3. "I use a broad except to make sure the service never crashes"**

This is actually **more dangerous** than crashing! By catching all exceptions and returning None:
- You can't distinguish "user not found" from "database down" from "typo in my code"
- Bugs are silently hidden instead of loudly failing
- The calling code can't decide how to handle different errors

**4. "I don't need @wraps because it's not a decorator"**

Correct! @wraps is specifically for decorators to preserve metadata. Your understanding here is right—I just want to confirm you know *why* decorators need it (they replace the function object, losing the original's `__name__` and `__doc__`).

---

## Security Issues (Critical!)

**SQL Injection Vulnerability:**
```python
# Your code:
self.db_connection.query(f"SELECT * FROM users WHERE id = {user_id}")

# Attack: user_id = "1; DROP TABLE users;--"
# Executes: SELECT * FROM users WHERE id = 1; DROP TABLE users;--
```

Always use parameterized queries:
```python
self.db_connection.query("SELECT * FROM users WHERE id = %s", (user_id,))
```

The same vulnerability exists in `create_user` with name and email.

---

## Corrected Code

```python
from typing import Optional
from dataclasses import dataclass

# Dataclass IS appropriate for data structures
@dataclass
class User:
    id: int
    name: str
    email: str

# Custom exceptions with context
class UserServiceError(Exception):
    """Base exception for UserService."""
    pass

class UserNotFoundError(UserServiceError):
    def __init__(self, user_id: int):
        self.user_id = user_id
        super().__init__(f"User {user_id} not found")

class ValidationError(UserServiceError):
    def __init__(self, field: str, reason: str):
        self.field = field
        self.reason = reason
        super().__init__(f"Invalid {field}: {reason}")

class DatabaseError(UserServiceError):
    """Wrap database errors with context."""
    pass


# Regular class for service (NOT dataclass)
class UserService:
    def __init__(self, db_connection):
        self.db_connection = db_connection
        self._cache: dict[int, User] = {}  # Instance-specific!
    
    def get_user(self, user_id: int) -> User:
        """
        Get user by ID.
        
        Raises:
            UserNotFoundError: User doesn't exist
            DatabaseError: Database operation failed
        """
        if user_id in self._cache:
            return self._cache[user_id]
        
        try:
            # Parameterized query - safe from injection
            row = self.db_connection.query(
                "SELECT id, name, email FROM users WHERE id = %s", 
                (user_id,)
            )
        except Exception as e:
            raise DatabaseError(f"Failed to fetch user {user_id}") from e
        
        if row is None:
            raise UserNotFoundError(user_id)
        
        user = User(id=row['id'], name=row['name'], email=row['email'])
        self._cache[user_id] = user
        return user
    
    def create_user(self, name: str, email: str) -> User:
        """
        Create a new user.
        
        Raises:
            ValidationError: Invalid input
            DatabaseError: Database operation failed
        """
        # Explicit validation with clear errors
        if not name or len(name) < 2:
            raise ValidationError("name", "must be at least 2 characters")
        if "@" not in email or "." not in email:
            raise ValidationError("email", "must be valid email format")
        
        try:
            # Parameterized query - safe from injection
            result = self.db_connection.insert(
                "INSERT INTO users (name, email) VALUES (%s, %s) RETURNING id",
                (name, email)
            )
        except Exception as e:
            raise DatabaseError(f"Failed to create user") from e
        
        return User(id=result['id'], name=name, email=email)


# Usage with proper error handling
service = UserService(db_connection=get_db())

try:
    user = service.get_user(123)
    print(f"Found: {user.name}")
except UserNotFoundError as e:
    print(f"User {e.user_id} doesn't exist")
except DatabaseError as e:
    print(f"Database problem: {e}")
    # Maybe retry, alert ops, etc.
```

---

### Mental Model: When to Use What

```
┌─────────────────────────────────────────────────────┐
│              PATTERN SELECTION GUIDE                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  "Is it DATA or BEHAVIOR?"                          │
│                                                     │
│  DATA (use @dataclass):                             │
│  • User, Order, Config, Response                    │
│  • "It holds information"                           │
│  • Few/no methods beyond simple accessors           │
│                                                     │
│  BEHAVIOR (use regular class):                      │
│  • UserService, PaymentProcessor, EmailSender       │
│  • "It does things"                                 │
│  • Has dependencies (database, external services)   │
│                                                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  "What should happen on error?"                     │
│                                                     │
│  EXPECTED ERROR (catch specifically):               │
│  • User not found → UserNotFoundError               │
│  • Invalid input → ValidationError                  │
│  • Caller can handle it                             │
│                                                     │
│  UNEXPECTED ERROR (let it crash):                   │
│  • Typo in code                                     │
│  • Logic error                                      │
│  • Caller CAN'T handle it—it's a bug               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

Let me know if you have questions about any of this!