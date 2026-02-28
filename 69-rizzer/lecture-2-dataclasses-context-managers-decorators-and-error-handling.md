# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PATTERN RECOGNITION, NOT MEMORIZATION                          │
│  ─────────────────────────────────────                          │
│  Each pattern solves a specific problem. Show the problem       │
│  first, then the pattern becomes obvious.                       │
│                                                                 │
│  BUILD TOWARD FRAMEWORKS                                        │
│  ───────────────────────                                        │
│  Every pattern here appears in FastAPI/Pydantic:                │
│  • Dataclasses → Pydantic models                                │
│  • Decorators → @app.get, @app.post                             │
│  • Context managers → Database sessions                         │
│  • Exceptions → HTTPException                                   │
│                                                                 │
│  CONNECT TO LECTURE 1                                           │
│  ───────────────────                                            │
│  Type hints appear everywhere in this lecture.                  │
│  Students see immediate payoff from Lecture 1.                  │
│                                                                 │
│  PRACTICAL OVER THEORETICAL                                     │
│  ─────────────────────────                                      │
│  Focus on patterns used daily in backend code.                  │
│  Skip esoteric features they won't need.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                  PYTHON PATTERNS FOR BACKEND                    │
│                     (3-4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: DATACLASSES (45 min)                                   │
│  ├─ 1.1 The Problem: Boilerplate Hell                           │
│  ├─ 1.2 Basic Dataclass Syntax                                  │
│  ├─ 1.3 Field Defaults and Factories                            │
│  ├─ 1.4 Post-Init Processing                                    │
│  ├─ 1.5 Frozen Dataclasses (Immutability)                       │
│  └─ 1.6 Connection to Pydantic (Preview)                        │
│                                                                 │
│  PART 2: CONTEXT MANAGERS (45 min)                              │
│  ├─ 2.1 The Problem: Resources Need Cleanup                     │
│  ├─ 2.2 The With Statement                                      │
│  ├─ 2.3 How Context Managers Work                               │
│  ├─ 2.4 Creating Your Own (Class-Based)                         │
│  ├─ 2.5 Creating Your Own (contextlib)                          │
│  └─ 2.6 Real-World Backend Examples                             │
│                                                                 │
│  PART 3: DECORATORS DEMYSTIFIED (60 min)                        │
│  ├─ 3.1 The Problem: Adding Behavior Without Modification       │
│  ├─ 3.2 Functions Are Objects                                   │
│  ├─ 3.3 The Wrapper Pattern (Step by Step)                      │
│  ├─ 3.4 The @ Syntax (Syntactic Sugar)                          │
│  ├─ 3.5 Preserving Function Metadata                            │
│  ├─ 3.6 Decorators With Arguments                               │
│  └─ 3.7 Connection to FastAPI (Preview)                         │
│                                                                 │
│  PART 4: CUSTOM EXCEPTIONS (30 min)                             │
│  ├─ 4.1 The Problem: Generic Errors Are Useless                 │
│  ├─ 4.2 Python's Exception Hierarchy                            │
│  ├─ 4.3 Creating Custom Exceptions                              │
│  ├─ 4.4 Exception Hierarchies                                   │
│  └─ 4.5 Rich Exceptions (Adding Context)                        │
│                                                                 │
│  PART 5: ERROR HANDLING PATTERNS (45 min)                       │
│  ├─ 5.1 try/except/else/finally                                 │
│  ├─ 5.2 Catching Specific Exceptions                            │
│  ├─ 5.3 Re-Raising and Exception Chaining                       │
│  ├─ 5.4 Pattern: Fail Fast vs Graceful Degradation              │
│  └─ 5.5 Error Handling Decision Framework                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: DATACLASSES

## 1.1 The Problem: Boilerplate Hell

**Start with the pain. Show why we need dataclasses.**

```python
# Without dataclasses — the old way

class User:
    def __init__(self, id: int, username: str, email: str, is_active: bool = True):
        self.id = id
        self.username = username
        self.email = email
        self.is_active = is_active
    
    def __repr__(self):
        return f"User(id={self.id}, username={self.username}, email={self.email}, is_active={self.is_active})"
    
    def __eq__(self, other):
        if not isinstance(other, User):
            return False
        return (self.id == other.id and 
                self.username == other.username and 
                self.email == other.email and 
                self.is_active == other.is_active)
    
    def __hash__(self):
        return hash((self.id, self.username, self.email, self.is_active))
```

**Count the problems:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE BOILERPLATE PROBLEM                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  For a simple 4-field class, we had to write:                   │
│                                                                 │
│  ├─ __init__    → Assign every field manually                   │
│  ├─ __repr__    → String representation for debugging           │
│  ├─ __eq__      → Equality comparison                           │
│  └─ __hash__    → If we want to use in sets/dicts               │
│                                                                 │
│  That's ~25 lines for what's essentially "a thing with fields"  │
│                                                                 │
│  Problems:                                                      │
│  • Repetitive: Each field appears 4+ times                      │
│  • Error-prone: Easy to forget a field in __eq__ or __repr__    │
│  • Maintenance: Add a field = update 4 methods                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "You're building a backend with User, Order, Product, Address, Payment... Do you want to write this boilerplate 50 times?"

---

## 1.2 Basic Dataclass Syntax

**The solution: @dataclass decorator**

```python
from dataclasses import dataclass

@dataclass
class User:
    id: int
    username: str
    email: str
    is_active: bool = True
```

**That's it.** Four lines. You get `__init__`, `__repr__`, `__eq__`, and more for free.

```python
# All of this works automatically:

user = User(id=1, username="alice", email="alice@example.com")
print(user)  # User(id=1, username='alice', email='alice@example.com', is_active=True)

user2 = User(id=1, username="alice", email="alice@example.com")
print(user == user2)  # True (automatic __eq__)

# Type hints are enforced at editor level (connect to Lecture 1!)
user = User(id="not_an_int", ...)  # mypy catches this!
```

**What @dataclass generates:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 WHAT @dataclass GENERATES                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @dataclass                                                     │
│  class User:                     GENERATES:                     │
│      id: int                     __init__(self, id, ...)        │
│      username: str       to       __repr__(self)                │
│      email: str                  __eq__(self, other)            │
│      is_active: bool             (optionally more)              │
│                                                                 │
│                                                                 │
│  The decorator reads your class, sees the annotated fields,     │
│  and WRITES the methods for you at class definition time.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to Lecture 1 (Type Hints):**

> "Notice how we're using type hints here: `id: int`, `email: str`. These aren't just documentation — they're what the dataclass reads to know your fields. Type hints from Lecture 1 are now REQUIRED, not optional."

---

## 1.3 Field Defaults and Factories

**Simple defaults work as expected:**

```python
from dataclasses import dataclass

@dataclass
class Config:
    host: str = "localhost"
    port: int = 8000
    debug: bool = False

config = Config()  # Uses all defaults
print(config)  # Config(host='localhost', port=8000, debug=False)

config2 = Config(port=3000)  # Override just port
print(config2)  # Config(host='localhost', port=3000, debug=False)
```

**But there's a trap with mutable defaults:**

```python
# ❌ WRONG — This is a bug!
@dataclass
class BadUser:
    name: str
    tags: list[str] = []  # ← SHARED between all instances!

user1 = BadUser("alice")
user2 = BadUser("bob")
user1.tags.append("admin")
print(user2.tags)  # ['admin'] — Wait, what?! Bob is admin too!
```

**The problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE MUTABLE DEFAULT TRAP                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Default values are evaluated ONCE at class definition time.    │
│                                                                 │
│      tags: list = []                                            │
│                    │                                            │
│                    ▼                                            │
│              ONE list object                                    │
│                    │                                            │
│          ┌─────────┴────────┐                                   │
│          ▼                  ▼                                   │
│       user1.tags       user2.tags   ← Same object!              │
│                                                                 │
│  This is actually a Python gotcha, not just dataclasses.        │
│  But dataclasses have a solution: field(default_factory=...)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The solution: field() with default_factory**

```python
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    tags: list[str] = field(default_factory=list)  # ✅ New list per instance
    metadata: dict[str, str] = field(default_factory=dict)  # ✅ New dict per instance

user1 = User("alice")
user2 = User("bob")
user1.tags.append("admin")
print(user2.tags)  # [] — Correct! Bob has his own list
```

**When to use field():**

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Order:
    id: int
    customer_id: int
    
    # Mutable defaults → use field(default_factory=...)
    items: list[str] = field(default_factory=list)
    
    # Exclude from __repr__ (e.g., sensitive data)
    internal_notes: str = field(default="", repr=False)
    
    # Exclude from comparison (timestamps shouldn't affect equality)
    created_at: Optional[str] = field(default=None, compare=False)
```

---

## 1.4 Post-Init Processing

**Sometimes you need to compute values after initialization:**

```python
from dataclasses import dataclass, field

@dataclass
class Rectangle:
    width: float
    height: float
    area: float = field(init=False)  # Don't include in __init__
    
    def __post_init__(self):
        """Called automatically after __init__"""
        self.area = self.width * self.height

rect = Rectangle(width=10, height=5)
print(rect)  # Rectangle(width=10, height=5, area=50)
print(rect.area)  # 50
```

**Real backend example — normalizing data:**

```python
@dataclass
class User:
    id: int
    email: str
    username: str = field(init=False)  # Derived from email
    
    def __post_init__(self):
        # Normalize email to lowercase
        self.email = self.email.lower().strip()
        # Extract username from email
        self.username = self.email.split("@")[0]

user = User(id=1, email="  ALICE@Example.com  ")
print(user.email)     # alice@example.com
print(user.username)  # alice
```

**__post_init__ flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    __post_init__ FLOW                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    User(id=1, email="ALICE@Example.com")                        │
│                       │                                         │
│                       ▼                                         │
│              Generated __init__                                 │
│              ├─ self.id = 1                                     │
│              └─ self.email = "ALICE@Example.com"                │
│                       │                                         │
│                       ▼                                         │
│              __post_init__()  ← Called automatically            │
│              ├─ self.email = self.email.lower().strip()         │
│              └─ self.username = self.email.split("@")[0]        │
│                       │                                         │
│                       ▼                                         │
│              Fully initialized object                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.5 Frozen Dataclasses (Immutability)

**Immutable objects are safer — they can't be accidentally modified:**

```python
@dataclass(frozen=True)
class Point:
    x: float
    y: float

point = Point(3.0, 4.0)
point.x = 5.0  # ❌ FrozenInstanceError: cannot assign to field 'x'
```

**Why immutability matters in backend code:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHY FROZEN DATACLASSES?                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. HASHABLE — Can be used as dict keys or in sets             │
│     ─────────                                                   │
│     @dataclass(frozen=True)                                     │
│     class Currency:                                             │
│         code: str                                               │
│         name: str                                               │
│                                                                 │
│     rates = {Currency("USD", "Dollar"): 1.0}  # ✅ Works!       │
│                                                                 │
│  2. THREAD-SAFE — No race conditions from mutation              │
│     ───────────                                                 │
│     Multiple async tasks can share frozen objects safely        │
│                                                                 │
│  3. PREDICTABLE — Value never changes after creation            │
│     ───────────                                                 │
│     Configuration objects, API responses, cached data           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Common pattern — configuration objects:**

```python
@dataclass(frozen=True)
class DatabaseConfig:
    host: str
    port: int
    database: str
    username: str
    password: str
    
    @property
    def connection_string(self) -> str:
        return f"postgresql://{self.username}:{self.password}@{self.host}:{self.port}/{self.database}"

# Config is set once, never changes
config = DatabaseConfig(
    host="localhost",
    port=5432,
    database="myapp",
    username="admin",
    password="secret"
)

# Cannot accidentally modify
config.host = "production.db.com"  # ❌ FrozenInstanceError
```

---

## 1.6 Connection to Pydantic (Preview)

**Dataclasses are the foundation. Pydantic builds on them.**

```
┌─────────────────────────────────────────────────────────────────┐
│              DATACLASS → PYDANTIC EVOLUTION                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DATACLASS (what you learned today)                             │
│  ──────────────────────────────────                             │
│  @dataclass                                                     │
│  class User:                                                    │
│      id: int                                                    │
│      email: str                                                 │
│      is_active: bool = True                                     │
│                                                                 │
│  • Auto-generates __init__, __repr__, __eq__                    │
│  • Type hints are "hints" — not enforced at runtime             │
│  • No automatic validation                                      │
│                                                                 │
│                                                                 │
│  PYDANTIC (Week 2, Lecture 3)                                   │
│  ───────────────────────────                                    │
│  class User(BaseModel):                                         │
│      id: int                                                    │
│      email: EmailStr  # ← Special validated type                │
│      is_active: bool = True                                     │
│                                                                 │
│  • Everything dataclass does, PLUS:                             │
│  • Runtime validation (invalid email → error!)                  │
│  • JSON serialization/deserialization                           │
│  • Automatic API documentation                                  │
│                                                                 │
│  SAME SYNTAX, MORE POWER.                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The mental model:**

> "Think of Pydantic as 'dataclasses with superpowers.' The syntax is almost identical, but Pydantic validates data at runtime. Today you learn the foundation, Week 2 adds the superpowers."

---

# PART 2: CONTEXT MANAGERS

## 2.1 The Problem: Resources Need Cleanup

**Resources that must be cleaned up:**

```python
# ❌ PROBLEM: What if an error occurs before close()?

file = open("data.txt", "r")
data = file.read()
process(data)  # ← If this crashes, file.close() never runs!
file.close()
```

**The dangers:**

```
┌─────────────────────────────────────────────────────────────────┐
│                RESOURCES THAT NEED CLEANUP                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RESOURCE               │ IF NOT CLEANED UP                     │
│  ───────────────────────┼───────────────────────────────────    │
│  File handles           │ Data not flushed, file stays locked   │
│  Database connections   │ Connection pool exhausted             │
│  Network sockets        │ Port stays occupied                   │
│  Locks                  │ Other threads blocked forever         │
│  Temporary files        │ Disk fills up                         │
│                                                                 │
│  In backend development, you deal with ALL of these.            │
│  Leaked resources = server crashes, hard-to-debug bugs.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Manual cleanup is fragile:**

```python
# ❌ Manual try/finally — verbose and easy to forget

file = open("data.txt", "r")
try:
    data = file.read()
    process(data)  # Even if this crashes...
finally:
    file.close()   # ...this ALWAYS runs
```

---

## 2.2 The With Statement

**The `with` statement guarantees cleanup:**

```python
# ✅ With statement — clean and safe

with open("data.txt", "r") as file:
    data = file.read()
    process(data)

# file.close() called automatically, even if process() crashes!
```

**The pattern:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE WITH STATEMENT                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  with EXPRESSION as VARIABLE:                                   │
│      BODY                                                       │
│                                                                 │
│                                                                 │
│  What happens:                                                  │
│                                                                 │
│  1. EXPRESSION evaluated → returns "context manager"            │
│  2. Context manager's __enter__() called → result is VARIABLE   │
│  3. BODY executes                                               │
│  4. Context manager's __exit__() called → cleanup               │
│     (even if BODY raised an exception!)                         │
│                                                                 │
│                                                                 │
│      with open("file.txt") as f:                                │
│           │                   │                                 │
│           │          __enter__() returns file object            │
│           │                                                     │
│           ▼                                                     │
│      open("file.txt") returns a context manager                 │
│                                                                 │
│      # ... use f ...                                            │
│                                                                 │
│      # Leaving the block → __exit__() closes the file           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Multiple context managers:**

```python
# Managing multiple resources at once

with open("input.txt", "r") as infile, open("output.txt", "w") as outfile:
    data = infile.read()
    outfile.write(data.upper())

# Both files closed automatically!
```

---

## 2.3 How Context Managers Work

**A context manager is any object with `__enter__` and `__exit__` methods:**

```python
class FileManager:
    def __init__(self, filename: str, mode: str):
        self.filename = filename
        self.mode = mode
        self.file = None
    
    def __enter__(self):
        """Called when entering 'with' block"""
        print(f"Opening {self.filename}")
        self.file = open(self.filename, self.mode)
        return self.file  # This becomes the 'as' variable
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """Called when leaving 'with' block (always!)"""
        print(f"Closing {self.filename}")
        if self.file:
            self.file.close()
        # Return False to propagate exceptions, True to suppress
        return False

# Usage
with FileManager("test.txt", "w") as f:
    f.write("Hello!")
```

Output:
```
Opening test.txt
Closing test.txt
```

**The __exit__ parameters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    __exit__ PARAMETERS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  def __exit__(self, exc_type, exc_val, exc_tb):                 │
│                     │         │        │                        │
│                     │         │        └─ Traceback object      │
│                     │         └─ Exception value/message        │
│                     └─ Exception class (or None)                │
│                                                                 │
│                                                                 │
│  If no exception occurred:                                      │
│      exc_type = None                                            │
│      exc_val = None                                             │
│      exc_tb = None                                              │
│                                                                 │
│  If exception occurred:                                         │
│      exc_type = ValueError (the class)                          │
│      exc_val = ValueError("something went wrong")               │
│      exc_tb = <traceback object>                                │
│                                                                 │
│                                                                 │
│  Return value:                                                  │
│      False (or None) → Exception propagates (normal)            │
│      True → Exception is SUPPRESSED (use carefully!)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Creating Your Own (Class-Based)

**Real backend example — timing operations:**

```python
import time
from typing import Optional

class Timer:
    """Context manager for timing code blocks"""
    
    def __init__(self, name: str = "Operation"):
        self.name = name
        self.start_time: Optional[float] = None
        self.elapsed: Optional[float] = None
    
    def __enter__(self) -> "Timer":
        self.start_time = time.time()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb) -> bool:
        self.elapsed = time.time() - self.start_time
        status = "failed" if exc_type else "completed"
        print(f"{self.name} {status} in {self.elapsed:.3f}s")
        return False  # Don't suppress exceptions

# Usage
with Timer("Database query"):
    time.sleep(0.5)  # Simulate slow operation

# Output: Database query completed in 0.501s

# Access elapsed time after block
with Timer("API call") as timer:
    time.sleep(0.2)

print(f"Took {timer.elapsed:.3f} seconds")  # Took 0.200 seconds
```

**Real backend example — database transaction:**

```python
class Transaction:
    """Context manager for database transactions"""
    
    def __init__(self, connection):
        self.connection = connection
    
    def __enter__(self):
        self.connection.begin()
        return self.connection
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            # No exception → commit
            self.connection.commit()
            print("Transaction committed")
        else:
            # Exception occurred → rollback
            self.connection.rollback()
            print(f"Transaction rolled back due to: {exc_val}")
        return False  # Propagate the exception

# Usage (pseudo-code)
with Transaction(db_connection) as conn:
    conn.execute("INSERT INTO users ...")
    conn.execute("INSERT INTO audit_log ...")
    # If any INSERT fails, BOTH are rolled back
```

---

## 2.5 Creating Your Own (contextlib)

**For simpler cases, use the `@contextmanager` decorator:**

```python
from contextlib import contextmanager
from typing import Generator

@contextmanager
def timer(name: str = "Operation") -> Generator[None, None, None]:
    """Simple timing context manager"""
    start = time.time()
    try:
        yield  # This is where the 'with' block runs
    finally:
        elapsed = time.time() - start
        print(f"{name} took {elapsed:.3f}s")

# Usage — identical to class-based version
with timer("Data processing"):
    time.sleep(0.3)
```

**How @contextmanager works:**

```
┌─────────────────────────────────────────────────────────────────┐
│               @contextmanager STRUCTURE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @contextmanager                                                │
│  def my_context():                                              │
│      # SETUP CODE         ← Runs on __enter__                   │
│      print("Before")                                            │
│                                                                 │
│      try:                                                       │
│          yield value      ← value becomes 'as' variable         │
│                           ← 'with' block runs HERE              │
│      finally:                                                   │
│          # CLEANUP CODE   ← Runs on __exit__ (always)           │
│          print("After")                                         │
│                                                                 │
│                                                                 │
│  The yield SPLITS your function into __enter__ and __exit__     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**More practical examples:**

```python
from contextlib import contextmanager
from typing import Generator
import os

@contextmanager
def change_directory(path: str) -> Generator[None, None, None]:
    """Temporarily change working directory"""
    original = os.getcwd()
    try:
        os.chdir(path)
        yield
    finally:
        os.chdir(original)

# Usage
print(os.getcwd())  # /home/user/project
with change_directory("/tmp"):
    print(os.getcwd())  # /tmp
print(os.getcwd())  # /home/user/project (restored!)


@contextmanager  
def temporary_env_var(key: str, value: str) -> Generator[None, None, None]:
    """Temporarily set an environment variable"""
    original = os.environ.get(key)
    try:
        os.environ[key] = value
        yield
    finally:
        if original is None:
            del os.environ[key]
        else:
            os.environ[key] = original

# Usage — great for testing
with temporary_env_var("DEBUG", "true"):
    # Code here sees DEBUG=true
    pass
# DEBUG restored to original value
```

---

## 2.6 Real-World Backend Examples

**Examples you'll see in Weeks 2-3:**

```
┌─────────────────────────────────────────────────────────────────┐
│            CONTEXT MANAGERS IN BACKEND CODE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DATABASE SESSIONS (Week 3)                                     │
│  ───────────────────────────                                    │
│  with Session() as session:                                     │
│      user = session.query(User).first()                         │
│      session.commit()                                           │
│  # Session automatically closed                                 │
│                                                                 │
│  HTTP CLIENTS (Week 4)                                          │
│  ─────────────────────                                          │
│  async with httpx.AsyncClient() as client:                      │
│      response = await client.get("https://api.example.com")     │
│  # Connection pool automatically cleaned up                     │
│                                                                 │
│  FILE UPLOADS                                                   │
│  ────────────                                                   │
│  with tempfile.NamedTemporaryFile() as tmp:                     │
│      tmp.write(uploaded_data)                                   │
│      process_file(tmp.name)                                     │
│  # Temp file automatically deleted                              │
│                                                                 │
│  LOCKS (threading)                                              │
│  ─────────────────                                              │
│  with threading.Lock():                                         │
│      shared_resource.modify()                                   │
│  # Lock automatically released                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Async context managers (preview for Lecture 3):**

```python
# You'll see this pattern with async code

class AsyncResource:
    async def __aenter__(self):
        """Async setup"""
        await self.connect()
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        """Async cleanup"""
        await self.disconnect()
        return False

# Usage with 'async with'
async with AsyncResource() as resource:
    await resource.do_something()
```

---

# PART 3: DECORATORS DEMYSTIFIED

## 3.1 The Problem: Adding Behavior Without Modification

**Scenario: You want to log every function call**

```python
# ❌ Without decorators — modify every function

def get_user(user_id: int) -> dict:
    print(f"Calling get_user with {user_id}")  # Added logging
    result = database.fetch_user(user_id)
    print(f"get_user returned {result}")       # Added logging
    return result

def get_orders(user_id: int) -> list:
    print(f"Calling get_orders with {user_id}")  # Copy-paste
    result = database.fetch_orders(user_id)
    print(f"get_orders returned {result}")       # Copy-paste
    return result

# Problem: Logging code duplicated in EVERY function
# Adding timing? Modify EVERY function again.
```

**What we want:**

```python
# ✅ With decorators — modify ONCE, apply ANYWHERE

@log_calls
def get_user(user_id: int) -> dict:
    return database.fetch_user(user_id)

@log_calls
def get_orders(user_id: int) -> list:
    return database.fetch_orders(user_id)

# Logging behavior defined ONCE, applied to any function
```

---

## 3.2 Functions Are Objects

**Before decorators, understand this: functions are objects.**

```python
def greet(name: str) -> str:
    return f"Hello, {name}!"

# Functions are objects — they can be:

# 1. Assigned to variables
say_hello = greet
print(say_hello("Alice"))  # Hello, Alice!

# 2. Stored in data structures
functions = [greet, len, print]

# 3. Passed as arguments
def call_twice(func, arg):
    func(arg)
    func(arg)

call_twice(print, "Hi!")  # Prints "Hi!" twice

# 4. Returned from other functions
def get_greeter():
    def inner_greet(name):
        return f"Hey, {name}!"
    return inner_greet

greeter = get_greeter()
print(greeter("Bob"))  # Hey, Bob!
```

**The key insight:**

```
┌─────────────────────────────────────────────────────────────────┐
│                FUNCTIONS ARE FIRST-CLASS OBJECTS                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  In Python, functions are just objects like integers or lists.  │
│                                                                 │
│      def greet(name):         greet ──▶ [function object]       │
│          return f"Hi, {name}"              │                    │
│                                            │                    │
│                                   ┌────────┴────────┐           │
│                                   │  Has attributes │           │
│                                   │  • __name__     │           │
│                                   │  • __doc__      │           │
│                                   │  Can be called  │           │
│                                   │  with ()        │           │
│                                   └─────────────────┘           │
│                                                                 │
│  This means we can:                                             │
│  • Pass functions to other functions                            │
│  • Return functions from functions                              │
│  • Create functions that MODIFY other functions                 │
│                                                                 │
│  That last one is what decorators do.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 The Wrapper Pattern (Step by Step)

**Step 1: A function that takes a function**

```python
def log_calls(func):
    """Takes a function, returns a 'wrapped' version"""
    
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__} with args={args}, kwargs={kwargs}")
        result = func(*args, **kwargs)  # Call the ORIGINAL function
        print(f"{func.__name__} returned {result}")
        return result
    
    return wrapper
```

**Step 2: Use it manually**

```python
def add(a: int, b: int) -> int:
    return a + b

# Wrap the function manually
logged_add = log_calls(add)

# Call the wrapped version
result = logged_add(3, 5)
```

Output:
```
Calling add with args=(3, 5), kwargs={}
add returned 8
```

**What happened:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE WRAPPER PATTERN                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  BEFORE:    logged_add = log_calls(add)                         │
│                                                                 │
│      add ─────────────▶ [original function]                     │
│                                                                 │
│                                                                 │
│  AFTER:                                                         │
│                                                                 │
│      logged_add ──────▶ [wrapper function]                      │
│                              │                                  │
│                              │ (contains reference to)          │
│                              ▼                                  │
│                         [original add]                          │
│                                                                 │
│                                                                 │
│  When you call logged_add(3, 5):                                │
│                                                                 │
│      1. wrapper() runs                                          │
│      2. wrapper() prints "Calling..."                           │
│      3. wrapper() calls ORIGINAL add(3, 5)                      │
│      4. wrapper() prints "returned..."                          │
│      5. wrapper() returns the result                            │
│                                                                 │
│  The original function is "wrapped" with extra behavior.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 The @ Syntax (Syntactic Sugar)

**The @ syntax is just shorthand:**

```python
# These two are EXACTLY equivalent:

# Without @ syntax
def add(a: int, b: int) -> int:
    return a + b
add = log_calls(add)  # Replace add with wrapped version

# With @ syntax
@log_calls
def add(a: int, b: int) -> int:
    return a + b
```

**The @ is applied at definition time:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    @ SYNTAX TRANSLATION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What you write:              What Python does:                 │
│  ────────────────             ─────────────────                 │
│                                                                 │
│  @decorator                   def func():                       │
│  def func():                      pass                          │
│      pass                     func = decorator(func)            │
│                                                                 │
│                                                                 │
│  @decorator1                  def func():                       │
│  @decorator2                      pass                          │
│  def func():                  func = decorator2(func)           │
│      pass                     func = decorator1(func)           │
│                               # Bottom decorator applied first! │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Complete example:**

```python
import time
from typing import Callable, Any

def timer(func: Callable) -> Callable:
    """Decorator that times function execution"""
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - start
        print(f"{func.__name__} took {elapsed:.4f} seconds")
        return result
    return wrapper

@timer
def slow_function() -> str:
    time.sleep(0.5)
    return "done"

result = slow_function()  # Prints: slow_function took 0.5001 seconds
```

---

## 3.5 Preserving Function Metadata

**There's a problem with our wrapper:**

```python
@timer
def slow_function() -> str:
    """This function is slow on purpose."""
    time.sleep(0.5)
    return "done"

print(slow_function.__name__)  # wrapper  ← Wrong! Should be "slow_function"
print(slow_function.__doc__)   # None     ← Wrong! Should be the docstring
```

**The solution: functools.wraps**

```python
from functools import wraps
from typing import Callable, Any

def timer(func: Callable) -> Callable:
    """Decorator that times function execution"""
    
    @wraps(func)  # ← This copies metadata from func to wrapper
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - start
        print(f"{func.__name__} took {elapsed:.4f} seconds")
        return result
    
    return wrapper

@timer
def slow_function() -> str:
    """This function is slow on purpose."""
    time.sleep(0.5)
    return "done"

print(slow_function.__name__)  # slow_function ✅
print(slow_function.__doc__)   # This function is slow on purpose. ✅
```

**Always use @wraps:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ALWAYS USE @wraps                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  from functools import wraps                                    │
│                                                                 │
│  def my_decorator(func):                                        │
│      @wraps(func)  # ← Always add this!                         │
│      def wrapper(*args, **kwargs):                              │
│          # ... your decorator logic ...                         │
│          return func(*args, **kwargs)                           │
│      return wrapper                                             │
│                                                                 │
│                                                                 │
│  @wraps copies these attributes from the original function:     │
│  • __name__       → Function name                               │
│  • __doc__        → Docstring                                   │
│  • __module__     → Module where defined                        │
│  • __annotations__→ Type hints!                                 │
│  • __dict__       → Function attributes                         │
│                                                                 │
│  This matters for:                                              │
│  • Debugging (stack traces show real function name)             │
│  • Documentation (help() shows real docstring)                  │
│  • Type checking (mypy sees real type hints)                    │
│  • FastAPI (needs metadata for API docs!)                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.6 Decorators With Arguments

**Sometimes you want to configure the decorator:**

```python
# We want this:
@retry(max_attempts=3)
def fetch_data():
    ...

# Not this (no customization):
@retry
def fetch_data():
    ...
```

**The trick: A function that returns a decorator**

```python
from functools import wraps
from typing import Callable, Any
import time

def retry(max_attempts: int = 3, delay: float = 1.0) -> Callable:
    """Decorator factory that returns a retry decorator"""
    
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            last_exception = None
            
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    print(f"Attempt {attempt} failed: {e}")
                    if attempt < max_attempts:
                        time.sleep(delay)
            
            raise last_exception
        
        return wrapper
    
    return decorator

# Usage
@retry(max_attempts=3, delay=0.5)
def unstable_api_call() -> str:
    import random
    if random.random() < 0.7:
        raise ConnectionError("API unavailable")
    return "Success!"

result = unstable_api_call()  # Retries up to 3 times
```

**How it works:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DECORATOR WITH ARGUMENTS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @retry(max_attempts=3)                                         │
│  def func():                                                    │
│      pass                                                       │
│                                                                 │
│  Is equivalent to:                                              │
│                                                                 │
│  def func():                                                    │
│      pass                                                       │
│  decorator = retry(max_attempts=3)   # Returns decorator        │
│  func = decorator(func)              # Apply to function        │
│                                                                 │
│                                                                 │
│  THREE LEVELS OF NESTING:                                       │
│                                                                 │
│  retry(max_attempts=3)  ──▶  decorator(func)  ──▶  wrapper()    │
│        │                          │                    │        │
│        │                          │                    │        │
│    Returns               Returns the              Actually      │
│    decorator             wrapper                  runs          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.7 Connection to FastAPI (Preview)

**Decorators are EVERYWHERE in FastAPI:**

```
┌─────────────────────────────────────────────────────────────────┐
│               DECORATORS IN FASTAPI (Week 2)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  from fastapi import FastAPI                                    │
│  app = FastAPI()                                                │
│                                                                 │
│  @app.get("/users/{user_id}")    ← Decorator!                   │
│  def get_user(user_id: int):                                    │
│      return {"user_id": user_id}                                │
│                                                                 │
│  @app.post("/users")             ← Decorator!                   │
│  def create_user(user: User):                                   │
│      return user                                                │
│                                                                 │
│                                                                 │
│  What these decorators do:                                      │
│  • Register the function as a route handler                     │
│  • Define the HTTP method (GET, POST, etc.)                     │
│  • Define the URL path                                          │
│  • Extract path parameters, query params, body                  │
│  • Validate input (using Pydantic)                              │
│  • Generate API documentation                                   │
│                                                                 │
│  Now you understand what's happening behind @app.get!           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The mental model:**

> "When you see `@app.get('/users')` in FastAPI, you now know: it's a decorator that takes your function, wraps it with request handling, validation, and routing logic, and registers it with the app. The @ syntax is the same you learned today."

---

# PART 4: CUSTOM EXCEPTIONS

## 4.1 The Problem: Generic Errors Are Useless

**Imagine debugging this:**

```python
# Somewhere deep in your code
raise Exception("Something went wrong")

# In production logs:
# Exception: Something went wrong

# Questions you can't answer:
# - What KIND of thing went wrong?
# - Can we retry?
# - Should we show this to the user?
# - Is this a bug or expected behavior?
```

**Compare to:**

```python
raise PaymentDeclinedError(
    user_id=123,
    amount=99.99,
    reason="insufficient_funds"
)

# Now you know:
# - It's a payment error (not a database error or API error)
# - Which user (123)
# - What amount (99.99)
# - Why it failed (insufficient funds)
# - How to handle it (show user-friendly message, don't retry)
```

---

## 4.2 Python's Exception Hierarchy

**Understanding the built-in structure:**

```
┌─────────────────────────────────────────────────────────────────┐
│              PYTHON'S EXCEPTION HIERARCHY                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BaseException                                                  │
│  ├── SystemExit          (sys.exit() called)                    │
│  ├── KeyboardInterrupt   (Ctrl+C pressed)                       │
│  ├── GeneratorExit       (generator closed)                     │
│  │                                                              │
│  └── Exception           ← Your exceptions inherit from HERE    │
│      ├── StopIteration                                          │
│      ├── ArithmeticError                                        │
│      │   ├── ZeroDivisionError                                  │
│      │   └── OverflowError                                      │
│      ├── LookupError                                            │
│      │   ├── KeyError                                           │
│      │   └── IndexError                                         │
│      ├── OSError                                                │
│      │   ├── FileNotFoundError                                  │
│      │   ├── PermissionError                                    │
│      │   └── ConnectionError                                    │
│      ├── ValueError                                             │
│      ├── TypeError                                              │
│      └── RuntimeError                                           │
│                                                                 │
│  RULE: Always inherit from Exception, not BaseException         │
│        (KeyboardInterrupt should NOT be caught!)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why hierarchy matters:**

```python
# You can catch at different levels of specificity

try:
    result = data[key]
except KeyError:
    # Only KeyError
    pass

try:
    result = data[key]
except LookupError:
    # KeyError OR IndexError (both are LookupError)
    pass

try:
    result = some_operation()
except Exception:
    # Almost any error (but not Ctrl+C!)
    pass
```

---

## 4.3 Creating Custom Exceptions

**The simplest custom exception:**

```python
class UserNotFoundError(Exception):
    """Raised when a user is not found in the database"""
    pass

# Usage
def get_user(user_id: int) -> dict:
    user = database.find_user(user_id)
    if user is None:
        raise UserNotFoundError(f"User {user_id} not found")
    return user
```

**Adding context (the useful pattern):**

```python
class UserNotFoundError(Exception):
    """Raised when a user is not found in the database"""
    
    def __init__(self, user_id: int, message: str = None):
        self.user_id = user_id
        self.message = message or f"User {user_id} not found"
        super().__init__(self.message)

# Usage
try:
    user = get_user(123)
except UserNotFoundError as e:
    print(f"User ID: {e.user_id}")  # Access structured data!
    log_error("user_not_found", user_id=e.user_id)
```

---

## 4.4 Exception Hierarchies

**Build a hierarchy for your application:**

```python
# Base exception for your application
class AppError(Exception):
    """Base exception for all application errors"""
    pass

# Domain-specific categories
class DatabaseError(AppError):
    """Base for all database-related errors"""
    pass

class APIError(AppError):
    """Base for all external API errors"""
    pass

class ValidationError(AppError):
    """Base for all validation errors"""
    pass

# Specific errors
class ConnectionPoolExhaustedError(DatabaseError):
    """No available database connections"""
    pass

class RecordNotFoundError(DatabaseError):
    """Requested record doesn't exist"""
    def __init__(self, table: str, id: int):
        self.table = table
        self.id = id
        super().__init__(f"{table} with id={id} not found")

class RateLimitExceededError(APIError):
    """External API rate limit hit"""
    def __init__(self, service: str, retry_after: int):
        self.service = service
        self.retry_after = retry_after
        super().__init__(f"{service} rate limited, retry after {retry_after}s")
```

**Visual hierarchy:**

```
┌─────────────────────────────────────────────────────────────────┐
│              APPLICATION EXCEPTION HIERARCHY                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AppError (base)                                                │
│  │                                                              │
│  ├── DatabaseError                                              │
│  │   ├── ConnectionPoolExhaustedError                           │
│  │   ├── RecordNotFoundError                                    │
│  │   └── QueryTimeoutError                                      │
│  │                                                              │
│  ├── APIError                                                   │
│  │   ├── RateLimitExceededError                                 │
│  │   ├── ServiceUnavailableError                                │
│  │   └── InvalidResponseError                                   │
│  │                                                              │
│  └── ValidationError                                            │
│      ├── InvalidEmailError                                      │
│      ├── InvalidAmountError                                     │
│      └── MissingFieldError                                      │
│                                                                 │
│                                                                 │
│  Benefits:                                                      │
│  • catch DatabaseError → catches ALL database errors            │
│  • catch RecordNotFoundError → catches only that specific one   │
│  • catch AppError → catches everything YOUR app raises          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 Rich Exceptions (Adding Context)

**Real backend example — API errors with all context:**

```python
from dataclasses import dataclass
from typing import Optional, Any
from datetime import datetime

@dataclass
class APIErrorContext:
    """Structured context for API errors"""
    service: str
    endpoint: str
    status_code: Optional[int]
    response_body: Optional[Any]
    timestamp: datetime
    request_id: Optional[str]

class ExternalAPIError(Exception):
    """Error from external API with full context"""
    
    def __init__(
        self, 
        message: str, 
        context: APIErrorContext,
        retryable: bool = False
    ):
        self.message = message
        self.context = context
        self.retryable = retryable
        super().__init__(message)
    
    def __str__(self) -> str:
        return (
            f"{self.message} "
            f"[service={self.context.service}, "
            f"endpoint={self.context.endpoint}, "
            f"status={self.context.status_code}]"
        )

# Usage
try:
    response = external_api.call("/prices")
except ExternalAPIError as e:
    if e.retryable:
        # Schedule retry
        queue_retry(e.context.service, e.context.endpoint)
    else:
        # Log and alert
        log_error(
            "external_api_failure",
            service=e.context.service,
            status_code=e.context.status_code,
            request_id=e.context.request_id
        )
```

---

# PART 5: ERROR HANDLING PATTERNS

## 5.1 try/except/else/finally

**The full structure (most people don't know about `else`):**

```python
try:
    # Code that might raise an exception
    result = risky_operation()
except SpecificError as e:
    # Handle specific exception
    handle_error(e)
except AnotherError as e:
    # Handle different exception
    handle_other_error(e)
else:
    # Runs ONLY if no exception occurred
    # Great for code that should run on success
    process_result(result)
finally:
    # ALWAYS runs (cleanup code)
    cleanup()
```

**Flow diagram:**

```
┌─────────────────────────────────────────────────────────────────┐
│              try/except/else/finally FLOW                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                         try block                               │
│                             │                                   │
│              ┌──────────────┴──────────────┐                    │
│              │                             │                    │
│        Exception raised              No exception               │
│              │                             │                    │
│              ▼                             ▼                    │
│       except block(s)                 else block                │
│              │                             │                    │
│              └──────────────┬──────────────┘                    │
│                             │                                   │
│                             ▼                                   │
│                       finally block                             │
│                      (ALWAYS runs)                              │
│                                                                 │
│                                                                 │
│  Why use else?                                                  │
│  • Code in else only runs if try succeeded                      │
│  • Keeps try block minimal (only risky code)                    │
│  • Makes success path explicit                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical example:**

```python
def read_config(path: str) -> dict:
    file = None
    try:
        file = open(path, "r")
        content = file.read()
    except FileNotFoundError:
        print(f"Config file not found: {path}")
        return {}
    except PermissionError:
        print(f"Permission denied: {path}")
        return {}
    else:
        # Only runs if file was read successfully
        return json.loads(content)
    finally:
        # Always close the file
        if file:
            file.close()
```

---

## 5.2 Catching Specific Exceptions

**Rule: Catch the most specific exception possible**

```python
# ❌ BAD: Catches everything, hides bugs
try:
    user = get_user(user_id)
    orders = get_orders(user.id)
    total = calculate_total(orders)
except Exception as e:
    return None  # Swallows ALL errors — even programming bugs!

# ❌ BAD: Bare except (even worse)
try:
    user = get_user(user_id)
except:  # Catches EVERYTHING including KeyboardInterrupt!
    pass

# ✅ GOOD: Specific exceptions
try:
    user = get_user(user_id)
except UserNotFoundError:
    return {"error": "User not found"}, 404
except DatabaseConnectionError:
    return {"error": "Service unavailable"}, 503
# Let other exceptions (bugs) propagate up
```

**Catching multiple exceptions:**

```python
# Multiple except blocks (different handling)
try:
    data = fetch_data()
except ConnectionError as e:
    log_error("Connection failed", error=e)
    return cached_data()  # Fallback
except TimeoutError as e:
    log_error("Request timeout", error=e)
    raise ServiceUnavailableError() from e

# Same handling for multiple types
try:
    data = fetch_data()
except (ConnectionError, TimeoutError) as e:
    # Same handling for both
    return cached_data()
```

---

## 5.3 Re-Raising and Exception Chaining

**Re-raising the current exception:**

```python
try:
    process_payment(amount)
except PaymentError as e:
    log_error("Payment failed", error=e)
    notify_admin(e)
    raise  # Re-raise the SAME exception with original traceback
```

**Exception chaining (from):**

```python
# Show the cause of an exception

class OrderError(Exception):
    pass

try:
    user = get_user(user_id)
except UserNotFoundError as e:
    # Chain exceptions: OrderError was CAUSED BY UserNotFoundError
    raise OrderError(f"Cannot create order for user {user_id}") from e
```

Output:
```
UserNotFoundError: User 123 not found

The above exception was the direct cause of the following exception:

OrderError: Cannot create order for user 123
```

**When to chain vs re-raise:**

```
┌─────────────────────────────────────────────────────────────────┐
│               RE-RAISE VS CHAIN VS WRAP                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  raise                                                          │
│  ─────                                                          │
│  Re-raise the same exception. Use when:                         │
│  • You logged the error but want it to propagate                │
│  • You did partial cleanup but can't handle it                  │
│                                                                 │
│  raise NewError() from original                                 │
│  ──────────────────────────────                                 │
│  Chain to a new exception type. Use when:                       │
│  • Translating low-level errors to domain errors                │
│  • Adding context while preserving cause                        │
│                                                                 │
│  raise NewError()  (no from)                                    │
│  ─────────────────                                              │
│  Replace exception. Use when:                                   │
│  • Original exception has sensitive info                        │
│  • Internal details shouldn't leak to callers                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 Pattern: Fail Fast vs Graceful Degradation

**Two valid strategies — know when to use each:**

```
┌─────────────────────────────────────────────────────────────────┐
│          FAIL FAST VS GRACEFUL DEGRADATION                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FAIL FAST                                                      │
│  ─────────                                                      │
│  Crash immediately when something unexpected happens.           │
│                                                                 │
│  When to use:                                                   │
│  • Development and testing (find bugs early)                    │
│  • Configuration errors (app can't run correctly anyway)        │
│  • Data corruption risks (better to stop than corrupt more)     │
│  • Critical operations (payment processing, medical records)    │
│                                                                 │
│  def connect_database():                                        │
│      try:                                                       │
│          return Database(config.db_url)                         │
│      except ConnectionError:                                    │
│          # FAIL FAST — app cannot work without database         │
│          raise SystemExit("Cannot connect to database!")        │
│                                                                 │
│                                                                 │
│  GRACEFUL DEGRADATION                                           │
│  ────────────────────                                           │
│  Continue with reduced functionality when possible.             │
│                                                                 │
│  When to use:                                                   │
│  • Non-critical features (recommendations, analytics)           │
│  • External services that might be slow/unavailable             │
│  • User-facing operations (better partial result than error)    │
│                                                                 │
│  def get_prices():                                              │
│      try:                                                       │
│          return fetch_from_api()                                │
│      except APIError:                                           │
│          # GRACEFUL — return cached data instead                │
│          return get_cached_prices()                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Example: Multi-source data fetching**

```python
from typing import Optional

async def get_crypto_price(symbol: str) -> Optional[float]:
    """Fetch price with graceful degradation across sources"""
    
    # Try primary source
    try:
        return await binance_client.get_price(symbol)
    except ExternalAPIError as e:
        log_warning("Binance unavailable", error=e)
    
    # Try backup source
    try:
        return await coinbase_client.get_price(symbol)
    except ExternalAPIError as e:
        log_warning("Coinbase unavailable", error=e)
    
    # Try cache as last resort
    cached = await cache.get(f"price:{symbol}")
    if cached:
        log_info("Using cached price", symbol=symbol, age=cached.age)
        return cached.value
    
    # All sources failed — now we fail
    raise PriceUnavailableError(symbol)
```

---

## 5.5 Error Handling Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│            ERROR HANDLING DECISION FRAMEWORK                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    An error occurred                            │
│                           │                                     │
│                           ▼                                     │
│               ┌───────────────────────┐                         │
│               │ Can I handle it here? │                         │
│               └───────────┬───────────┘                         │
│                    │             │                              │
│                   YES            NO                             │
│                    │             │                              │
│                    ▼             ▼                              │
│            ┌─────────────┐  ┌─────────────┐                     │
│            │ Do I have   │  │ Propagate   │                     │
│            │ a fallback? │  │ (let caller │                     │
│            └──────┬──────┘  │ handle it)  │                     │
│               │       │     └─────────────┘                     │
│              YES      NO                                        │
│               │       │                                         │
│               ▼       ▼                                         │
│         ┌────────┐ ┌────────┐                                   │
│         │ Use    │ │ Log    │                                   │
│         │ fallb- │ │ and    │                                   │
│         │ ack    │ │ return │                                   │
│         │        │ │ error  │                                   │
│         └────────┘ └────────┘                                   │
│                                                                 │
│                                                                 │
│  LOGGING RULES:                                                 │
│  • Always log errors before handling or re-raising              │
│  • Include context: what operation, what input, what failed     │
│  • Use structured logging (key=value) not just messages         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│               PYTHON PATTERNS QUICK REFERENCE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DATACLASSES:                                                   │
│      from dataclasses import dataclass, field                   │
│                                                                 │
│      @dataclass                                                 │
│      class User:                                                │
│          id: int                                                │
│          name: str                                              │
│          tags: list[str] = field(default_factory=list)          │
│                                                                 │
│      @dataclass(frozen=True)  # Immutable                       │
│      class Config:                                              │
│          host: str                                              │
│                                                                 │
│  CONTEXT MANAGERS:                                              │
│      with open("file.txt") as f:                                │
│          data = f.read()                                        │
│                                                                 │
│      @contextmanager                                            │
│      def my_context():                                          │
│          setup()                                                │
│          try:                                                   │
│              yield value                                        │
│          finally:                                               │
│              cleanup()                                          │
│                                                                 │
│  DECORATORS:                                                    │
│      from functools import wraps                                │
│                                                                 │
│      def my_decorator(func):                                    │
│          @wraps(func)                                           │
│          def wrapper(*args, **kwargs):                          │
│              # before                                           │
│              result = func(*args, **kwargs)                     │
│              # after                                            │
│              return result                                      │
│          return wrapper                                         │
│                                                                 │
│  CUSTOM EXCEPTIONS:                                             │
│      class AppError(Exception):                                 │
│          pass                                                   │
│                                                                 │
│      class NotFoundError(AppError):                             │
│          def __init__(self, resource: str, id: int):            │
│              self.resource = resource                           │
│              self.id = id                                       │
│              super().__init__(f"{resource} {id} not found")     │
│                                                                 │
│  ERROR HANDLING:                                                │
│      try:                                                       │
│          risky_operation()                                      │
│      except SpecificError as e:                                 │
│          handle_error(e)                                        │
│      else:                                                      │
│          on_success()                                           │
│      finally:                                                   │
│          cleanup()                                              │
│                                                                 │
│      raise NewError("message") from original_error              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  FOUR PATTERNS YOU'LL USE DAILY                                 │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  DATACLASSES                                            │    │
│  │  Structured data without boilerplate.                   │    │
│  │  Foundation for Pydantic (Week 2).                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                         │                                       │
│                         ▼                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  CONTEXT MANAGERS                                       │    │
│  │  Guaranteed resource cleanup.                           │    │
│  │  "with" = setup + body + teardown.                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                         │                                       │
│                         ▼                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  DECORATORS                                             │    │
│  │  Add behavior without modifying functions.              │    │
│  │  Understand @syntax before FastAPI.                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                         │                                       │
│                         ▼                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  EXCEPTION HIERARCHIES                                  │    │
│  │  Structured error handling.                             │    │
│  │  Rich context for debugging.                            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  These patterns appear in EVERY backend framework.              │
│  Master them now, recognize them everywhere.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Upcoming Lectures

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LECTURE 3 (Async Fundamentals):                                │
│  └─ async context managers (async with)                         │
│     Exception handling works the same in async code             │
│                                                                 │
│  LECTURE 4 (Developer Workflow):                                │
│  └─ Context managers for temporary directories in tests         │
│     Custom exceptions for CLI errors                            │
│                                                                 │
│  WEEK 1 MINI-PROJECT:                                           │
│  └─ Custom exceptions for API failures                          │
│     Dataclasses for price data                                  │
│     Decorators for retry logic                                  │
│                                                                 │
│  WEEK 2 (FastAPI):                                              │
│  └─ Pydantic models (evolved dataclasses)                       │
│     @app.get decorators (now you understand them!)              │
│     HTTPException (custom exception pattern)                    │
│     Dependency injection (context manager pattern)              │
│                                                                 │
│  WEEK 3 (Database):                                             │
│  └─ SQLAlchemy sessions as context managers                     │
│     Database-specific exceptions                                │
│                                                                 │
│  WEEK 4 (External APIs):                                        │
│  └─ httpx.AsyncClient as async context manager                  │
│     Retry decorators for unreliable services                    │
│     Exception hierarchies for API errors                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---
