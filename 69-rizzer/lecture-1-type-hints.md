# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BUGS FIRST, TYPES SECOND                                       │
│  ────────────────────────                                       │
│  Students must SEE the bugs that types prevent.                 │
│  Show the crash, then show the cure.                            │
│                                                                 │
│  PROGRESSIVE COMPLEXITY                                         │
│  ──────────────────────                                         │
│  Basic hints → Generics → Protocols → Narrowing                 │
│  Each layer builds on the previous.                             │
│                                                                 │
│  CONTRACTS, NOT CONSTRAINTS                                     │
│  ─────────────────────────                                      │
│  Frame types as "documentation that the computer checks"        │
│  not as "rules that slow you down."                             │
│                                                                 │
│  REAL-WORLD PATTERNS                                            │
│  ───────────────────                                            │
│  Every concept connects to backend development patterns         │
│  they'll use in FastAPI, SQLAlchemy, etc.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                    TYPE HINTS DEEP DIVE                         │
│                     (3-4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (30 min)                                   │
│  ├─ 1.1 The Bug That Could Have Been Caught                     │
│  ├─ 1.2 What Are Type Hints?                                    │
│  ├─ 1.3 Type Hints ≠ Type Enforcement                           │
│  └─ 1.4 The Tooling Ecosystem                                   │
│                                                                 │
│  PART 2: FOUNDATIONS (45 min)                                   │
│  ├─ 2.1 Basic Type Annotations                                  │
│  ├─ 2.2 The typing Module                                       │
│  ├─ 2.3 Optional and Union                                      │
│  ├─ 2.4 Collections with Type Parameters                        │
│  └─ 2.5 Callable Types                                          │
│                                                                 │
│  PART 3: GENERICS (60 min)                                      │
│  ├─ 3.1 The Problem: Flexible Yet Typed                         │
│  ├─ 3.2 TypeVar — The Type Placeholder                          │
│  ├─ 3.3 Generic Functions                                       │
│  ├─ 3.4 Generic Classes                                         │
│  ├─ 3.5 Bounded TypeVars                                        │
│  └─ 3.6 Real-World Pattern: Generic Repository                  │
│                                                                 │
│  PART 4: PROTOCOLS (45 min)                                     │
│  ├─ 4.1 The Problem: Duck Typing Without Safety                 │
│  ├─ 4.2 What Is Structural Subtyping?                           │
│  ├─ 4.3 Defining Protocols                                      │
│  ├─ 4.4 Protocols vs Abstract Base Classes                      │
│  └─ 4.5 Real-World Pattern: Pluggable Services                  │
│                                                                 │
│  PART 5: TYPE NARROWING (45 min)                                │
│  ├─ 5.1 The Problem: Union Types Need Refinement                │
│  ├─ 5.2 isinstance() Narrowing                                  │
│  ├─ 5.3 Custom Type Guards                                      │
│  ├─ 5.4 Assertion-Based Narrowing                               │
│  └─ 5.5 Pattern Matching (Python 3.10+)                         │
│                                                                 │
│  PART 6: PUTTING IT ALL TOGETHER (15 min)                       │
│  ├─ 6.1 When to Use What                                        │
│  └─ 6.2 Common Mistakes                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Bug That Could Have Been Caught

**Start with a real bug. Make them feel the pain.**

```python
# bug_demo.py — Run this with students watching

def calculate_total_price(items, discount):
    """Calculate total price with discount applied."""
    total = sum(item["price"] * item["quantity"] for item in items)
    return total * (1 - discount)

def send_order_confirmation(user, order_id):
    """Send confirmation email to user."""
    print(f"Sending email to {user['email']} for order {order_id}")

# Somewhere else in the codebase, 6 months later...
# A new developer writes:

cart = [
    {"price": 29.99, "quantity": 2},
    {"price": 49.99, "quantity": 1},
]

# Bug 1: Passed percentage instead of decimal
total = calculate_total_price(cart, 20)  # Meant 20% = 0.20
print(f"Total: ${total}")  # Outputs: $-2099.79 (!!)

# Bug 2: Passed user_id instead of user dict
send_order_confirmation(12345, "ORD-001")  # TypeError at runtime!
```

**Run it. Watch it crash or produce nonsense.**

```
Total: $-2099.79
Traceback (most recent call last):
  File "bug_demo.py", line 19, in <module>
    send_order_confirmation(12345, "ORD-001")
  File "bug_demo.py", line 8, in send_order_confirmation
    print(f"Sending email to {user['email']} for order {order_id}")
TypeError: 'int' object is not subscriptable
```

**Now ask the class:** 

> "This code is syntactically valid Python. It ran. It crashed in production at 2 AM. The person who wrote `calculate_total_price` left the company. How could we have prevented this?"

---

**Now show the typed version:**

```python
# typed_version.py
from typing import TypedDict

class CartItem(TypedDict):
    price: float
    quantity: int

class User(TypedDict):
    id: int
    email: str
    name: str

def calculate_total_price(items: list[CartItem], discount: float) -> float:
    """
    Calculate total price with discount applied.
    
    Args:
        items: List of cart items
        discount: Discount as decimal (0.20 = 20%)
    """
    total = sum(item["price"] * item["quantity"] for item in items)
    return total * (1 - discount)

def send_order_confirmation(user: User, order_id: str) -> None:
    """Send confirmation email to user."""
    print(f"Sending email to {user['email']} for order {order_id}")
```

**Now the type checker catches it BEFORE runtime:**

```
$ mypy typed_version.py

error: Argument 2 to "calculate_total_price" has incompatible type 
       "int"; expected "float" (but semantically, 20 vs 0.20 is still 
       a logic bug — types help but aren't magic)
       
error: Argument 1 to "send_order_confirmation" has incompatible type 
       "int"; expected "User"
```

---

## 1.2 What Are Type Hints?

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHAT ARE TYPE HINTS?                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Type hints are ANNOTATIONS that describe:                      │
│  ├─ What type a variable should hold                            │
│  ├─ What types a function accepts                               │
│  └─ What type a function returns                                │
│                                                                 │
│                                                                 │
│  def greet(name: str) -> str:                                   │
│              ▲    ▲       ▲                                     │
│              │    │       │                                     │
│              │    │       └─ Return type annotation             │
│              │    │                                             │
│              │    └─ Parameter type annotation                  │
│              │                                                  │
│              └─ Parameter name                                  │
│                                                                 │
│                                                                 │
│  age: int = 25                                                  │
│       ▲                                                         │
│       │                                                         │
│       └─ Variable type annotation                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight:**

> "Type hints are like a CONTRACT. The function signature says: 'I promise to take this and return that.' The type checker verifies that everyone honors their contracts."

---

## 1.3 Type Hints ≠ Type Enforcement

**Critical: Python does NOT enforce types at runtime!**

```python
def greet(name: str) -> str:
    return f"Hello, {name}"

# Python happily runs this — no error!
result = greet(12345)
print(result)  # "Hello, 12345"
```

```
┌─────────────────────────────────────────────────────────────────┐
│              TYPE HINTS ARE NOT TYPE ENFORCEMENT                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Python at runtime:                                             │
│  ┌─────────────────────────────────────────┐                    │
│  │ "You wrote 'str' but passed an int?     │                    │
│  │  Cool, I don't care. Running it anyway."│                    │
│  └─────────────────────────────────────────┘                    │
│                                                                 │
│  Type checker (mypy, pyright):                                  │
│  ┌─────────────────────────────────────────┐                    │
│  │ "You wrote 'str' but passed an int?     │                    │
│  │  ERROR! Fix this before you ship."      │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                 │
│                                                                 │
│  TYPE HINTS ARE:                  TYPE HINTS ARE NOT:           │
│  ─────────────────                ───────────────────           │
│  ✓ Documentation                  ✗ Runtime enforcement         │
│  ✓ Checked by external tools      ✗ Performance optimization    │
│  ✓ IDE autocomplete fuel          ✗ Required to run Python      │
│  ✓ Bug prevention (pre-runtime)   ✗ A guarantee code works      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 The Tooling Ecosystem

**Type hints are useless without tools that read them:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE TYPE CHECKING ECOSYSTEM                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOUR CODE                                                      │
│  (with type hints)                                              │
│       │                                                         │
│       ├──────────────────┬──────────────────┐                   │
│       ▼                  ▼                  ▼                   │
│  ┌─────────┐        ┌─────────┐        ┌─────────┐              │
│  │  mypy   │        │ pyright │        │  IDE    │              │
│  │         │        │         │        │ (VS Code│              │
│  │ Static  │        │ Faster, │        │ PyCharm)│              │
│  │ checker │        │ stricter│        │         │              │
│  └─────────┘        └─────────┘        └─────────┘              │
│       │                  │                  │                   │
│       ▼                  ▼                  ▼                   │
│  Errors in            Errors in         Red squiggles          │
│  terminal             terminal          as you type             │
│                                                                 │
│                                                                 │
│  RUNTIME TOOLS (Optional):                                      │
│  ├─ pydantic: Validates at runtime (FastAPI uses this!)        │
│  ├─ beartype: Fast runtime checking                            │
│  └─ typeguard: Comprehensive runtime checking                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to run mypy:**

```bash
# Install
pip install mypy

# Check a file
mypy my_file.py

# Check a directory
mypy src/

# Strict mode (catches more issues)
mypy --strict src/
```

---

# PART 2: FOUNDATIONS

## 2.1 Basic Type Annotations

**The syntax you'll use 90% of the time:**

```python
# Variable annotations
name: str = "Alice"
age: int = 30
price: float = 19.99
is_active: bool = True

# Function annotations
def greet(name: str) -> str:
    return f"Hello, {name}"

def add(a: int, b: int) -> int:
    return a + b

def process(data: bytes) -> None:  # Returns nothing
    print(data)
```

**The basic types:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      BASIC TYPES                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TYPE        EXAMPLE VALUES              NOTES                  │
│  ────        ──────────────              ─────                  │
│  str         "hello", 'world', ""        Text                   │
│  int         42, -1, 0                   Whole numbers          │
│  float       3.14, -0.5, 1.0             Decimal numbers        │
│  bool        True, False                 Boolean                │
│  bytes       b"hello", b'\x00'           Binary data            │
│  None        None                        Absence of value       │
│                                                                 │
│  IMPORTANT:                                                     │
│  • int is NOT a subtype of float in Python's type system       │
│  • Use float if you want to accept both int and float          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 The typing Module

**For complex types, import from `typing`:**

```python
from typing import Any, Optional, Union, Literal

# Any — Disables type checking for this value (escape hatch)
def log_anything(data: Any) -> None:
    print(data)

# Literal — Specific values only
def set_status(status: Literal["pending", "active", "done"]) -> None:
    print(f"Status: {status}")

set_status("active")   # ✅ OK
set_status("invalid")  # ❌ Error: not a valid literal
```

**Python 3.9+ built-in generic types:**

```python
# Before Python 3.9:
from typing import List, Dict, Set, Tuple

def old_style(items: List[str]) -> Dict[str, int]:
    ...

# Python 3.9+ (use this!):
def new_style(items: list[str]) -> dict[str, int]:
    ...

# The lowercase versions are preferred now
```

---

## 2.3 Optional and Union

**When a value can be multiple types:**

```python
from typing import Optional, Union

# Union — Can be any of these types
def parse_id(value: Union[str, int]) -> int:
    if isinstance(value, str):
        return int(value)
    return value

# Python 3.10+ shorthand for Union
def parse_id_modern(value: str | int) -> int:
    if isinstance(value, str):
        return int(value)
    return value

# Optional — Can be the type OR None
# Optional[X] is shorthand for Union[X, None]
def find_user(user_id: int) -> Optional[dict]:
    if user_id == 1:
        return {"id": 1, "name": "Alice"}
    return None  # Not found

# Python 3.10+ shorthand
def find_user_modern(user_id: int) -> dict | None:
    ...
```

**Visualizing Optional:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      OPTIONAL[X]                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Optional[str] means: "This is EITHER a str OR None"            │
│                                                                 │
│  ┌─────────────────────────┐                                    │
│  │     Optional[str]       │                                    │
│  │  ┌─────────┬─────────┐  │                                    │
│  │  │   str   │  None   │  │                                    │
│  │  │ "hello" │  None   │  │                                    │
│  │  │ "world" │         │  │                                    │
│  │  └─────────┴─────────┘  │                                    │
│  └─────────────────────────┘                                    │
│                                                                 │
│  Common use case: Function that might not find something        │
│                                                                 │
│  def get_user(id: int) -> Optional[User]:                       │
│      # Returns User if found, None if not found                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Collections with Type Parameters

**Specify what's INSIDE the collection:**

```python
# Lists — ordered, mutable
def get_names() -> list[str]:
    return ["Alice", "Bob", "Charlie"]

def sum_prices(prices: list[float]) -> float:
    return sum(prices)

# Dictionaries — key-value pairs
def get_user_ages() -> dict[str, int]:
    return {"Alice": 30, "Bob": 25}

def count_words(text: str) -> dict[str, int]:
    words = text.split()
    return {word: words.count(word) for word in set(words)}

# Sets — unordered, unique values
def get_unique_tags(articles: list[dict]) -> set[str]:
    tags: set[str] = set()
    for article in articles:
        tags.update(article.get("tags", []))
    return tags

# Tuples — fixed length, possibly mixed types
def get_coordinates() -> tuple[float, float]:
    return (40.7128, -74.0060)

def get_user_info() -> tuple[str, int, bool]:
    return ("Alice", 30, True)  # name, age, is_active

# Variable-length tuple of same type
def get_scores() -> tuple[int, ...]:
    return (95, 87, 92, 88)
```

**Nested collections:**

```python
# List of dictionaries
def get_users() -> list[dict[str, str | int]]:
    return [
        {"name": "Alice", "age": 30},
        {"name": "Bob", "age": 25}
    ]

# Dictionary with list values
def get_tags_by_category() -> dict[str, list[str]]:
    return {
        "tech": ["python", "api", "backend"],
        "finance": ["trading", "crypto", "stocks"]
    }
```

---

## 2.5 Callable Types

**When a parameter is a function:**

```python
from typing import Callable

# A function that takes no args, returns str
def get_greeter() -> Callable[[], str]:
    def greet():
        return "Hello!"
    return greet

# A function that takes (int, int), returns int
def apply_operation(
    a: int, 
    b: int, 
    operation: Callable[[int, int], int]
) -> int:
    return operation(a, b)

# Usage:
def add(x: int, y: int) -> int:
    return x + y

def multiply(x: int, y: int) -> int:
    return x * y

result = apply_operation(5, 3, add)       # 8
result = apply_operation(5, 3, multiply)  # 15
```

**Callable syntax explained:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    CALLABLE SYNTAX                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Callable[[ParamType1, ParamType2, ...], ReturnType]            │
│           ▲                               ▲                     │
│           │                               │                     │
│           │                               └─ What it returns    │
│           │                                                     │
│           └─ List of parameter types                            │
│                                                                 │
│                                                                 │
│  EXAMPLES:                                                      │
│  ─────────                                                      │
│  Callable[[], None]              No params, returns nothing     │
│  Callable[[str], int]            Takes str, returns int         │
│  Callable[[int, int], float]     Takes two ints, returns float  │
│  Callable[..., str]              Any params, returns str        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: GENERICS

## 3.1 The Problem: Flexible Yet Typed

**Consider this function:**

```python
def first_element(items: list) -> ???:
    """Return the first element of a list."""
    if items:
        return items[0]
    return None
```

**What's the return type?**

```python
# If we pass list[str], we want str back
name = first_element(["Alice", "Bob"])  # Should be str

# If we pass list[int], we want int back
number = first_element([1, 2, 3])  # Should be int

# If we pass list[User], we want User back
user = first_element(users)  # Should be User
```

**The problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      THE PROBLEM                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Option 1: Use `Any`                                            │
│  ───────────────────                                            │
│  def first(items: list[Any]) -> Any:                            │
│      ...                                                        │
│                                                                 │
│  Problem: We lose all type information!                         │
│  result = first(["hello"])                                      │
│  result.upper()  # Type checker: "Any can be anything" 🤷       │
│                                                                 │
│                                                                 │
│  Option 2: Write separate functions for each type               │
│  ─────────────────────────────────────────────────              │
│  def first_str(items: list[str]) -> str | None: ...             │
│  def first_int(items: list[int]) -> int | None: ...             │
│  def first_user(items: list[User]) -> User | None: ...          │
│                                                                 │
│  Problem: Infinite duplication! 😱                              │
│                                                                 │
│                                                                 │
│  Option 3: GENERICS                                             │
│  ──────────────────                                             │
│  One function that PRESERVES the type relationship.             │
│  "Whatever type goes in, that type comes out."                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 TypeVar — The Type Placeholder

**`TypeVar` is a placeholder that says "some type, TBD":**

```python
from typing import TypeVar

# Create a type variable
T = TypeVar("T")  # Name must match the variable name

def first_element(items: list[T]) -> T | None:
    """Return the first element, preserving the type."""
    if items:
        return items[0]
    return None
```

**How it works:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    HOW TypeVar WORKS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  T = TypeVar("T")                                               │
│                                                                 │
│  def first_element(items: list[T]) -> T | None:                 │
│                           ▲           ▲                         │
│                           │           │                         │
│                           └─────┬─────┘                         │
│                                 │                               │
│                      "Same T in both places"                    │
│                                                                 │
│                                                                 │
│  When you CALL the function, T gets "filled in":                │
│                                                                 │
│  first_element(["a", "b"])   →  T = str    →  returns str|None  │
│  first_element([1, 2, 3])    →  T = int    →  returns int|None  │
│  first_element([user1])      →  T = User   →  returns User|None │
│                                                                 │
│                                                                 │
│  The type checker REMEMBERS what T was:                         │
│                                                                 │
│  name = first_element(["Alice"])                                │
│  name.upper()  # ✅ Type checker knows name is str|None         │
│                                                                 │
│  number = first_element([1, 2])                                 │
│  number.upper()  # ❌ Error: int has no 'upper' method          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Generic Functions

**More examples of generic functions:**

```python
from typing import TypeVar

T = TypeVar("T")
K = TypeVar("K")  # Can have multiple TypeVars
V = TypeVar("V")

# Identity function — returns exactly what you pass
def identity(value: T) -> T:
    return value

# Swap — returns tuple with types reversed
def swap(pair: tuple[T, K]) -> tuple[K, T]:
    return (pair[1], pair[0])

# Usage:
result = swap(("hello", 42))  # Returns tuple[int, str]
text, number = result         # text: int, number: str

# Get or default — returns same type as default
def get_or_default(value: T | None, default: T) -> T:
    if value is None:
        return default
    return value

# Usage:
name = get_or_default(None, "Anonymous")  # Returns str
count = get_or_default(None, 0)           # Returns int
```

**Dictionary operations:**

```python
def get_values(d: dict[K, V], keys: list[K]) -> list[V]:
    """Get multiple values from a dictionary."""
    return [d[k] for k in keys if k in d]

# Usage:
ages: dict[str, int] = {"Alice": 30, "Bob": 25}
results = get_values(ages, ["Alice", "Charlie"])  # list[int]
```

---

## 3.4 Generic Classes

**Classes can also be generic:**

```python
from typing import TypeVar, Generic

T = TypeVar("T")

class Box(Generic[T]):
    """A container that holds one item of type T."""
    
    def __init__(self, item: T) -> None:
        self._item = item
    
    def get(self) -> T:
        return self._item
    
    def replace(self, new_item: T) -> T:
        old = self._item
        self._item = new_item
        return old

# Usage — type is specified when creating instance:
string_box: Box[str] = Box("hello")
value = string_box.get()  # Type checker knows: str

int_box: Box[int] = Box(42)
value = int_box.get()  # Type checker knows: int

# Type error:
string_box.replace(123)  # ❌ Error: expected str, got int
```

**Multiple type parameters:**

```python
from typing import TypeVar, Generic

K = TypeVar("K")
V = TypeVar("V")

class Pair(Generic[K, V]):
    """Holds two related values of potentially different types."""
    
    def __init__(self, key: K, value: V) -> None:
        self.key = key
        self.value = value
    
    def to_tuple(self) -> tuple[K, V]:
        return (self.key, self.value)
    
    def swap(self) -> "Pair[V, K]":
        return Pair(self.value, self.key)

# Usage:
entry: Pair[str, int] = Pair("age", 30)
print(entry.key)    # str
print(entry.value)  # int

swapped = entry.swap()  # Pair[int, str]
```

---

## 3.5 Bounded TypeVars

**Sometimes you need to constrain what types are allowed:**

```python
from typing import TypeVar

# Unbounded — any type allowed
T = TypeVar("T")

# Bounded — must be subclass of specified type
from typing import TypeVar

class Animal:
    def speak(self) -> str:
        return "..."

class Dog(Animal):
    def speak(self) -> str:
        return "Woof!"

class Cat(Animal):
    def speak(self) -> str:
        return "Meow!"

# T must be Animal or a subclass of Animal
AnimalType = TypeVar("AnimalType", bound=Animal)

def make_speak(animal: AnimalType) -> str:
    return animal.speak()  # ✅ Type checker knows .speak() exists

# Usage:
make_speak(Dog())    # ✅ OK
make_speak(Cat())    # ✅ OK
make_speak("hello")  # ❌ Error: str is not an Animal
```

**Constrained TypeVar — specific types only:**

```python
from typing import TypeVar

# Can ONLY be str or bytes, nothing else
StrOrBytes = TypeVar("StrOrBytes", str, bytes)

def concat(a: StrOrBytes, b: StrOrBytes) -> StrOrBytes:
    return a + b

concat("hello", " world")  # ✅ str
concat(b"hello", b"world") # ✅ bytes
concat(1, 2)               # ❌ Error: int not allowed
concat("hello", b"world")  # ❌ Error: can't mix str and bytes
```

---

## 3.6 Real-World Pattern: Generic Repository

**This pattern appears everywhere in backend development:**

```python
from typing import TypeVar, Generic, Optional
from dataclasses import dataclass

# Base model with ID
@dataclass
class BaseModel:
    id: int

# Specific models
@dataclass
class User(BaseModel):
    name: str
    email: str

@dataclass
class Product(BaseModel):
    name: str
    price: float

# Generic repository — works with any model type
ModelType = TypeVar("ModelType", bound=BaseModel)

class Repository(Generic[ModelType]):
    """Generic CRUD repository."""
    
    def __init__(self) -> None:
        self._storage: dict[int, ModelType] = {}
    
    def add(self, item: ModelType) -> ModelType:
        self._storage[item.id] = item
        return item
    
    def get(self, id: int) -> Optional[ModelType]:
        return self._storage.get(id)
    
    def get_all(self) -> list[ModelType]:
        return list(self._storage.values())
    
    def delete(self, id: int) -> bool:
        if id in self._storage:
            del self._storage[id]
            return True
        return False

# Usage — fully typed!
user_repo: Repository[User] = Repository()
product_repo: Repository[Product] = Repository()

# Type checker knows exactly what you're working with:
user_repo.add(User(id=1, name="Alice", email="alice@example.com"))
user = user_repo.get(1)  # Optional[User]

if user:
    print(user.email)  # ✅ Type checker knows user has .email

# This would be an error:
user_repo.add(Product(id=1, name="Widget", price=9.99))  # ❌ Wrong type!
```

---

# PART 4: PROTOCOLS

## 4.1 The Problem: Duck Typing Without Safety

**Python uses "duck typing" — if it walks like a duck...**

```python
def print_length(obj):
    """Print the length of anything that has a length."""
    print(f"Length: {len(obj)}")

# All of these work:
print_length([1, 2, 3])      # list
print_length("hello")         # str
print_length({"a": 1})        # dict
print_length((1, 2, 3))       # tuple

# This crashes at runtime:
print_length(42)              # TypeError: object of type 'int' has no len()
```

**The problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE DUCK TYPING PROBLEM                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  def process_items(container):                                  │
│      for item in container:    # Needs to be iterable           │
│          print(len(item))      # Items need __len__             │
│                                                                 │
│                                                                 │
│  Q: What TYPE should we annotate 'container' with?              │
│                                                                 │
│  Option 1: list[str]                                            │
│  └─ Too restrictive! Tuples, sets, custom classes also work.    │
│                                                                 │
│  Option 2: Any                                                  │
│  └─ Too permissive! We lose all type safety.                    │
│                                                                 │
│  Option 3: Create a base class, make everything inherit from it │
│  └─ Invasive! Can't modify built-in types or third-party code.  │
│                                                                 │
│  Option 4: PROTOCOLS ✅                                         │
│  └─ "Anything that has these methods" — non-invasive duck       │
│     typing with type safety.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 What Is Structural Subtyping?

**Two ways to check if types are compatible:**

```
┌─────────────────────────────────────────────────────────────────┐
│           NOMINAL VS STRUCTURAL SUBTYPING                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NOMINAL SUBTYPING (Inheritance-based)                          │
│  ─────────────────────────────────────                          │
│  "Is this class a declared subclass of that class?"             │
│                                                                 │
│  class Animal: ...                                              │
│  class Dog(Animal): ...  # Dog IS-A Animal (declared)           │
│                                                                 │
│  def pet(animal: Animal): ...                                   │
│  pet(Dog())  # ✅ OK, Dog inherits from Animal                  │
│                                                                 │
│                                                                 │
│  STRUCTURAL SUBTYPING (Shape-based)                             │
│  ──────────────────────────────────                             │
│  "Does this class have the required methods/attributes?"        │
│                                                                 │
│  class Duck:                                                    │
│      def quack(self): ...                                       │
│                                                                 │
│  class Robot:  # No inheritance relationship!                   │
│      def quack(self): ...                                       │
│                                                                 │
│  def make_noise(thing: CanQuack): ...                           │
│  make_noise(Duck())   # ✅ Has quack()                          │
│  make_noise(Robot())  # ✅ Has quack()                          │
│                                                                 │
│  PROTOCOLS enable structural subtyping in Python's type system  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Defining Protocols

**Use `Protocol` to define structural types:**

```python
from typing import Protocol

class Printable(Protocol):
    """Anything that can be converted to a string representation."""
    
    def to_string(self) -> str:
        """Return a string representation."""
        ...  # No implementation needed in Protocol

# These classes don't inherit from Printable, but they match it:
class User:
    def __init__(self, name: str):
        self.name = name
    
    def to_string(self) -> str:
        return f"User({self.name})"

class Product:
    def __init__(self, name: str, price: float):
        self.name = name
        self.price = price
    
    def to_string(self) -> str:
        return f"Product({self.name}, ${self.price})"

# Function accepts anything that matches Printable Protocol:
def log_item(item: Printable) -> None:
    print(f"Logging: {item.to_string()}")

# Both work, despite no inheritance relationship:
log_item(User("Alice"))           # ✅
log_item(Product("Widget", 9.99)) # ✅

# This fails — int doesn't have to_string():
log_item(42)  # ❌ Error: int doesn't implement Printable
```

**Protocol with multiple methods:**

```python
from typing import Protocol

class Serializable(Protocol):
    """Can be converted to/from JSON-compatible dict."""
    
    def to_dict(self) -> dict:
        ...
    
    @classmethod
    def from_dict(cls, data: dict) -> "Serializable":
        ...

class Repository(Protocol):
    """Basic CRUD operations."""
    
    def get(self, id: int) -> dict | None:
        ...
    
    def save(self, item: dict) -> int:
        ...
    
    def delete(self, id: int) -> bool:
        ...
```

---

## 4.4 Protocols vs Abstract Base Classes

**When to use which?**

```
┌─────────────────────────────────────────────────────────────────┐
│              PROTOCOL VS ABSTRACT BASE CLASS                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ABSTRACT BASE CLASS (ABC)                                      │
│  ─────────────────────────                                      │
│                                                                 │
│  from abc import ABC, abstractmethod                            │
│                                                                 │
│  class Drawable(ABC):                                           │
│      @abstractmethod                                            │
│      def draw(self) -> None: ...                                │
│                                                                 │
│  class Circle(Drawable):  # MUST inherit                        │
│      def draw(self) -> None:                                    │
│          print("Drawing circle")                                │
│                                                                 │
│  ✓ Enforced at class definition (can't instantiate ABC)         │
│  ✓ Can provide default implementations                          │
│  ✗ Requires explicit inheritance                                │
│  ✗ Can't use with third-party classes you don't control         │
│                                                                 │
│                                                                 │
│  PROTOCOL                                                       │
│  ────────                                                       │
│                                                                 │
│  from typing import Protocol                                    │
│                                                                 │
│  class Drawable(Protocol):                                      │
│      def draw(self) -> None: ...                                │
│                                                                 │
│  class Circle:  # No inheritance needed!                        │
│      def draw(self) -> None:                                    │
│          print("Drawing circle")                                │
│                                                                 │
│  ✓ No inheritance required                                      │
│  ✓ Works with existing classes (even built-ins)                 │
│  ✓ Better for defining interfaces/contracts                     │
│  ✗ Only checked by type checkers (not at runtime by default)    │
│                                                                 │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  • Protocol: When you want to accept "anything with method X"   │
│  • ABC: When you want a class hierarchy with shared code        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 Real-World Pattern: Pluggable Services

**This pattern is used extensively in backend systems:**

```python
from typing import Protocol
from dataclasses import dataclass

# Define what a cache must look like
class Cache(Protocol):
    """Protocol for cache implementations."""
    
    def get(self, key: str) -> str | None:
        """Get value by key, or None if not found."""
        ...
    
    def set(self, key: str, value: str, ttl: int = 300) -> None:
        """Set value with optional TTL in seconds."""
        ...
    
    def delete(self, key: str) -> bool:
        """Delete key, return True if existed."""
        ...

# Implementation 1: In-memory (for development)
class MemoryCache:
    """Simple in-memory cache."""
    
    def __init__(self) -> None:
        self._data: dict[str, str] = {}
    
    def get(self, key: str) -> str | None:
        return self._data.get(key)
    
    def set(self, key: str, value: str, ttl: int = 300) -> None:
        self._data[key] = value  # Ignoring TTL for simplicity
    
    def delete(self, key: str) -> bool:
        if key in self._data:
            del self._data[key]
            return True
        return False

# Implementation 2: Redis (for production) — same interface!
class RedisCache:
    """Redis-backed cache."""
    
    def __init__(self, host: str, port: int) -> None:
        self._host = host
        self._port = port
        # self._client = redis.Redis(host, port)
    
    def get(self, key: str) -> str | None:
        # return self._client.get(key)
        return None  # Placeholder
    
    def set(self, key: str, value: str, ttl: int = 300) -> None:
        # self._client.setex(key, ttl, value)
        pass
    
    def delete(self, key: str) -> bool:
        # return self._client.delete(key) > 0
        return False

# Service that uses any cache — doesn't care which implementation!
class UserService:
    def __init__(self, cache: Cache) -> None:
        self._cache = cache
    
    def get_user(self, user_id: int) -> dict | None:
        # Check cache first
        cached = self._cache.get(f"user:{user_id}")
        if cached:
            return {"id": user_id, "data": cached}
        
        # Fetch from database, cache result
        user_data = self._fetch_from_db(user_id)
        if user_data:
            self._cache.set(f"user:{user_id}", str(user_data))
        return user_data
    
    def _fetch_from_db(self, user_id: int) -> dict | None:
        # Database query here
        return {"id": user_id, "name": "Alice"}

# Easy to swap implementations:
dev_service = UserService(MemoryCache())
prod_service = UserService(RedisCache("localhost", 6379))
```

**The power of protocols:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROTOCOL BENEFITS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. TESTABILITY                                                 │
│     └─ Inject MemoryCache for tests, RedisCache for prod        │
│                                                                 │
│  2. FLEXIBILITY                                                 │
│     └─ Add new implementations without changing existing code   │
│                                                                 │
│  3. TYPE SAFETY                                                 │
│     └─ Type checker ensures all implementations are compatible  │
│                                                                 │
│  4. DOCUMENTATION                                               │
│     └─ Protocol is explicit contract: "must have these methods" │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: TYPE NARROWING

## 5.1 The Problem: Union Types Need Refinement

**When you have a Union type, the type checker is conservative:**

```python
def process(value: str | int) -> None:
    # Type checker only knows: value is str OR int
    
    print(value.upper())  # ❌ Error! int doesn't have .upper()
    print(value + 1)      # ❌ Error! str doesn't support + 1
    
    # We KNOW it's one or the other, but the type checker doesn't
    # know WHICH one at any given point in the code
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE NARROWING PROBLEM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  value: str | int                                               │
│                                                                 │
│  ┌─────────────────────────┐                                    │
│  │       str | int         │  ← Type checker sees this          │
│  │  ┌─────────┬─────────┐  │                                    │
│  │  │   str   │   int   │  │                                    │
│  │  │ .upper()│ + - * / │  │                                    │
│  │  │ .split()│ .bit_   │  │                                    │
│  │  └─────────┴─────────┘  │                                    │
│  └─────────────────────────┘                                    │
│                                                                 │
│  Problem: Type checker won't let us use str OR int methods      │
│  because it can't guarantee which type we have.                 │
│                                                                 │
│  Solution: TYPE NARROWING                                       │
│  Tell the type checker which type we have at each point.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 isinstance() Narrowing

**The most common way to narrow types:**

```python
def process(value: str | int) -> str:
    if isinstance(value, str):
        # Type checker KNOWS: value is str here
        return value.upper()  # ✅ OK!
    else:
        # Type checker KNOWS: value is int here
        return str(value * 2)  # ✅ OK!

# Works with multiple types
def handle_input(data: str | bytes | None) -> str:
    if data is None:
        return ""
    
    if isinstance(data, bytes):
        return data.decode("utf-8")
    
    # Only str remains
    return data
```

**Visualizing the narrowing:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  isinstance() NARROWING                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  def process(value: str | int | None) -> str:                   │
│      │                                                          │
│      │  value: str | int | None                                 │
│      ▼                                                          │
│      if value is None:                                          │
│          │                                                      │
│          │  value: None  ← Narrowed!                            │
│          │  return ""                                           │
│          │                                                      │
│      ▼  (value: str | int  ← None eliminated)                   │
│                                                                 │
│      if isinstance(value, str):                                 │
│          │                                                      │
│          │  value: str  ← Narrowed!                             │
│          │  return value.upper()                                │
│          │                                                      │
│      ▼  (value: int  ← str eliminated)                          │
│                                                                 │
│      return str(value)  # value is definitely int               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Multiple types with isinstance:**

```python
def serialize(value: str | int | float | list) -> str:
    # Check multiple types at once
    if isinstance(value, (int, float)):
        # value: int | float
        return f"number:{value}"
    
    if isinstance(value, str):
        # value: str
        return f"text:{value}"
    
    # value: list (only option left)
    return f"list:{len(value)} items"
```

---

## 5.3 Custom Type Guards

**For complex narrowing logic, define your own type guards:**

```python
from typing import TypeGuard

class User:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email

class Admin(User):
    def __init__(self, name: str, email: str, permissions: list[str]):
        super().__init__(name, email)
        self.permissions = permissions

def is_admin(user: User) -> TypeGuard[Admin]:
    """Check if user is an Admin."""
    return isinstance(user, Admin)

def handle_user(user: User) -> None:
    if is_admin(user):
        # Type checker KNOWS: user is Admin here
        print(f"Admin with permissions: {user.permissions}")  # ✅
    else:
        # user is still just User
        print(f"Regular user: {user.name}")
```

**TypeGuard with dict validation:**

```python
from typing import TypedDict, TypeGuard

class ValidResponse(TypedDict):
    status: str
    data: dict
    timestamp: int

def is_valid_response(obj: dict) -> TypeGuard[ValidResponse]:
    """Check if dict matches ValidResponse structure."""
    return (
        isinstance(obj, dict) and
        isinstance(obj.get("status"), str) and
        isinstance(obj.get("data"), dict) and
        isinstance(obj.get("timestamp"), int)
    )

def process_response(response: dict) -> None:
    if is_valid_response(response):
        # Type checker KNOWS: response is ValidResponse
        print(f"Status: {response['status']}")
        print(f"Timestamp: {response['timestamp']}")
    else:
        raise ValueError("Invalid response format")
```

---

## 5.4 Assertion-Based Narrowing

**`assert` statements also narrow types:**

```python
def process(value: str | None) -> str:
    assert value is not None  # Narrows type
    
    # Type checker KNOWS: value is str here
    return value.upper()  # ✅ OK

# More explicit version with error message
def get_user_name(user: dict | None) -> str:
    assert user is not None, "User must not be None"
    assert "name" in user, "User must have a name"
    
    return str(user["name"])
```

**Warning about assert:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ASSERT WARNING                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ⚠️  assert statements are REMOVED when Python runs with -O     │
│                                                                 │
│  python -O script.py  ← Optimized mode, no asserts!             │
│                                                                 │
│  For production code, prefer:                                   │
│  ├─ isinstance() checks with explicit handling                  │
│  ├─ if/raise patterns                                           │
│  └─ Type guards                                                 │
│                                                                 │
│  Use assert for:                                                │
│  ├─ Development/debugging                                       │
│  ├─ Test code                                                   │
│  └─ Invariants that "should never happen"                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 Pattern Matching (Python 3.10+)

**The `match` statement provides powerful narrowing:**

```python
def describe(value: int | str | list | None) -> str:
    match value:
        case None:
            # value: None
            return "Nothing here"
        
        case int():
            # value: int
            return f"Integer: {value}"
        
        case str():
            # value: str
            return f"String of length {len(value)}"
        
        case list():
            # value: list
            return f"List with {len(value)} items"
        
        case _:
            # Catch-all (shouldn't reach here with our type)
            return "Unknown"
```

**Pattern matching with destructuring:**

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

@dataclass
class Circle:
    center: Point
    radius: float

@dataclass
class Rectangle:
    top_left: Point
    width: float
    height: float

Shape = Point | Circle | Rectangle

def area(shape: Shape) -> float:
    match shape:
        case Point():
            # A point has no area
            return 0.0
        
        case Circle(radius=r):
            # Extract radius, calculate area
            return 3.14159 * r * r
        
        case Rectangle(width=w, height=h):
            # Extract dimensions, calculate area
            return w * h

# Usage:
print(area(Circle(Point(0, 0), 5)))      # 78.54
print(area(Rectangle(Point(0, 0), 4, 3))) # 12.0
```

**Matching literal values:**

```python
def http_status_message(code: int) -> str:
    match code:
        case 200:
            return "OK"
        case 201:
            return "Created"
        case 400:
            return "Bad Request"
        case 401:
            return "Unauthorized"
        case 404:
            return "Not Found"
        case 500:
            return "Internal Server Error"
        case _:
            return f"Unknown status: {code}"
```

---

# PART 6: PUTTING IT ALL TOGETHER

## 6.1 When to Use What

```
┌─────────────────────────────────────────────────────────────────┐
│                    DECISION GUIDE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "I need a function that works with multiple types"             │
│  ─────────────────────────────────────────────────              │
│  Q: Are the types related (same structure)?                     │
│  ├─ YES, same methods → Use PROTOCOL                            │
│  │   def process(item: Saveable) → any class with .save()       │
│  │                                                              │
│  └─ NO, different structures → Use UNION or OVERLOAD            │
│      def process(x: int | str) → handle each differently        │
│                                                                 │
│                                                                 │
│  "I need a function that preserves input type in output"        │
│  ───────────────────────────────────────────────────            │
│  Use GENERICS with TypeVar                                      │
│  def first(items: list[T]) -> T                                 │
│                                                                 │
│                                                                 │
│  "I need to handle None or missing values"                      │
│  ─────────────────────────────────────────                      │
│  Use OPTIONAL (Union with None)                                 │
│  def find(id: int) -> User | None                               │
│                                                                 │
│                                                                 │
│  "I need to work differently based on actual type"              │
│  ─────────────────────────────────────────────────              │
│  Use TYPE NARROWING                                             │
│  if isinstance(value, str): ...                                 │
│  match value: case str(): ...                                   │
│                                                                 │
│                                                                 │
│  "I need a container class that holds any type"                 │
│  ──────────────────────────────────────────────                 │
│  Use GENERIC CLASS                                              │
│  class Cache(Generic[T]): ...                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 Common Mistakes

### Mistake 1: Using Any when you mean Unknown

```python
# ❌ WRONG: Any disables type checking
def process(data: Any) -> Any:
    return data["key"]  # No error, but could crash!

# ✅ BETTER: Be specific about what you expect
def process(data: dict[str, str]) -> str:
    return data["key"]  # Type checker validates usage

# ✅ OR: Use object for truly unknown types
def log_anything(data: object) -> None:
    print(str(data))  # Can only use methods on object
```

---

### Mistake 2: Forgetting to narrow Union types

```python
def format_value(value: str | int) -> str:
    # ❌ WRONG: Type checker doesn't know which type
    return value.upper()  # Error!

# ✅ CORRECT: Narrow first
def format_value(value: str | int) -> str:
    if isinstance(value, str):
        return value.upper()
    return str(value)
```

---

### Mistake 3: Mutable default arguments with types

```python
# ❌ DANGEROUS: Mutable default shared across calls
def add_item(item: str, items: list[str] = []) -> list[str]:
    items.append(item)
    return items

# ✅ CORRECT: Use None and create new list
def add_item(item: str, items: list[str] | None = None) -> list[str]:
    if items is None:
        items = []
    items.append(item)
    return items
```

---

### Mistake 4: Confusing Generic class vs instance

```python
from typing import Generic, TypeVar

T = TypeVar("T")

class Box(Generic[T]):
    def __init__(self, item: T) -> None:
        self.item = item

# ❌ WRONG: Forgot to specify type parameter
def get_box() -> Box:  # Box of what?
    return Box("hello")

# ✅ CORRECT: Specify the type parameter
def get_box() -> Box[str]:
    return Box("hello")
```

---

### Mistake 5: Protocol doesn't match signature exactly

```python
from typing import Protocol

class Processor(Protocol):
    def process(self, data: str) -> int:
        ...

class MyProcessor:
    # ❌ WRONG: Different parameter name doesn't matter, but
    # different TYPES do!
    def process(self, data: bytes) -> int:  # bytes != str
        return len(data)

# MyProcessor does NOT implement Processor!
# The type checker will catch this.
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                TYPE HINTS QUICK REFERENCE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BASIC TYPES:                                                   │
│      str, int, float, bool, bytes, None                         │
│                                                                 │
│  COLLECTIONS (Python 3.9+):                                     │
│      list[str], dict[str, int], set[int], tuple[str, int]       │
│                                                                 │
│  OPTIONAL/UNION:                                                │
│      Optional[str]  =  str | None                               │
│      Union[str, int]  =  str | int                              │
│                                                                 │
│  GENERICS:                                                      │
│      T = TypeVar("T")                                           │
│      def first(items: list[T]) -> T | None: ...                 │
│                                                                 │
│  BOUNDED GENERICS:                                              │
│      T = TypeVar("T", bound=BaseClass)                          │
│      T = TypeVar("T", str, bytes)  # Only these types           │
│                                                                 │
│  GENERIC CLASSES:                                               │
│      class Box(Generic[T]):                                     │
│          def __init__(self, item: T) -> None: ...               │
│                                                                 │
│  PROTOCOLS:                                                     │
│      class Saveable(Protocol):                                  │
│          def save(self) -> None: ...                            │
│                                                                 │
│  TYPE GUARDS:                                                   │
│      def is_str(x: object) -> TypeGuard[str]:                   │
│          return isinstance(x, str)                              │
│                                                                 │
│  CALLABLE:                                                      │
│      Callable[[int, str], bool]  # Takes int, str; returns bool │
│                                                                 │
│  LITERAL:                                                       │
│      Literal["read", "write", "execute"]                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  TYPE HINTS = CONTRACTS + DOCUMENTATION + SAFETY                │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                     Your Code                           │    │
│  │                         │                               │    │
│  │    "I accept str, return int"                           │    │
│  │                         │                               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                    │
│           ┌────────────────┼────────────────┐                   │
│           ▼                ▼                ▼                   │
│    ┌────────────┐    ┌────────────┐    ┌───────────┐            │
│    │Type Checker│    │   IDE      │    │  Future   │            │
│    │  (mypy)    │    │Autocomplete│    │   You     │            │
│    │            │    │            │    │           │            │
│    │ Catches    │    │ Suggests   │    │Understands│            │
│    │ errors     │    │ methods    │    │ the code  │            │
│    └────────────┘    └────────────┘    └───────────┘            │
│                                                                 │
│                                                                 │
│  GENERICS: "Same type goes in and out"                          │
│  ├─ TypeVar("T") is a placeholder                               │
│  └─ Preserves type information through transformations          │
│                                                                 │
│  PROTOCOLS: "Anything with these methods"                       │
│  ├─ Structural subtyping (duck typing with safety)              │
│  └─ No inheritance required                                     │
│                                                                 │
│  TYPE NARROWING: "Now I know it's specifically this type"       │
│  ├─ isinstance() is the most common tool                        │
│  └─ Lets you use type-specific methods safely                   │
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
│  WEEK 1 (Next lectures):                                        │
│  ├─ Error Handling: Custom exceptions with typed hierarchies    │
│  └─ Async: Type hints work identically with async def           │
│                                                                 │
│  WEEK 2 (FastAPI):                                              │
│  ├─ Pydantic uses type hints for automatic validation           │
│  ├─ Path/Query params typed: def get_user(id: int)              │
│  └─ Response models: -> User returns typed JSON                 │
│                                                                 │
│  WEEK 3 (SQLAlchemy):                                           │
│  ├─ Typed ORM models                                            │
│  └─ Generic repository pattern (you learned it today!)          │
│                                                                 │
│  WEEK 4 (External APIs):                                        │
│  ├─ TypedDict for API responses                                 │
│  └─ Protocols for pluggable HTTP clients                        │
│                                                                 │
│  WEEK 10-11 (Trading System):                                   │
│  ├─ Protocol-based strategy pattern                             │
│  └─ Generic indicator calculations                              │
│                                                                 │
│                                                                 │
│  TYPE HINTS ARE EVERYWHERE IN MODERN PYTHON BACKEND DEV         │
│  Master them now — you'll use them in every lecture.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

