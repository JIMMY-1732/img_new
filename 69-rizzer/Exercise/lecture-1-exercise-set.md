# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "Type hints make Python check types when the code runs, just like Java or C++."

**Specific Misconceptions:**

1. **Runtime Enforcement Myth:** Students think `def greet(name: str)` will reject an integer at runtime. Actually, Python completely ignores type hints when executing code.
2. **TypeVar as Code Generation:** Students think `TypeVar` creates multiple versions of a function (like C++ templates). Actually, it's purely a static analysis tool that tracks type relationships.
3. **Protocol Requires Inheritance:** Students think classes must explicitly inherit from a Protocol to match it. Actually, structural subtyping means "has the methods" is sufficient.
4. **Union Auto-Dispatch:** Students think `Union[str, int]` means Python automatically handles both types. Actually, the type checker conservatively blocks operations valid for only one type until you narrow.
5. **Type Hints = Type Safety:** Students conflate "I wrote type hints" with "my code is type safe." Actually, hints without a type checker are just documentation.

**The Prediction Gap:** When given code with type hints but mismatched values, students expect the code to crash or refuse to run. The actual result is the code runs normally (possibly producing wrong results), while a separate type checker tool reports errors. This reveals that Python's type system is opt-in tooling, not language enforcement.

---

## The Main Purpose (Essence)

**Core Insight:** Type hints are machine-readable contracts that tools verify at development time—they catch bugs before you run the code, not while it runs.

**The Problem Being Solved:**
Without type hints, you discover type mismatches through runtime crashes (often in production at 2 AM). With type hints and a type checker, you discover them instantly in your editor before you even save the file.

**The Key Distinctions:**

| Concept | What It Actually Is | Common Confusion |
|---------|--------------------|--------------------|
| TypeVar | A placeholder that preserves type relationships across a signature | Template instantiation / code generation |
| Protocol | "Anything with these methods" (structural) | "Must inherit from this" (nominal) |
| Type Narrowing | Telling the checker "in this branch, I know it's specifically X" | Automatic type dispatch |

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|--------------|---------------------|-----------------|------------|
| Type hints enforce at runtime | `greet(123)` crashes when `name: str` | Code runs, prints "Hello, 123" | Python ignores annotations at runtime |
| TypeVar creates function copies | `first([1,2])` and `first(["a"])` call different functions | Same function called; type checker tracks relationship statically | TypeVar is analysis-only metadata |
| Protocol needs inheritance | Class without `(Protocol)` parent won't type-check | Class matches if it has required methods | Structural subtyping checks shape, not lineage |
| Union means auto-handling | `value.upper()` works on `str \| int` | Type checker error: int has no upper() | Checker sees union, not actual runtime type |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Runtime Surprise

```python
# runtime_surprise.py
# Run this code and observe what happens

def process_user(user_id: int, username: str) -> dict:
    """Process a user and return their profile."""
    return {
        "id": user_id,
        "name": username,
        "display": f"User #{user_id}: {username}"
    }

def calculate_discount(price: float, percentage: float) -> float:
    """Apply percentage discount to price."""
    return price * (1 - percentage / 100)

# --- Main execution ---
if __name__ == "__main__":
    # Call 1: Swapped arguments (string where int expected, int where string expected)
    result1 = process_user("alice", 12345)
    print(f"Result 1: {result1}")
    
    # Call 2: Completely wrong types
    result2 = process_user([1, 2, 3], {"name": "test"})
    print(f"Result 2: {result2}")
    
    # Call 3: String instead of float
    result3 = calculate_discount("fifty", "twenty")
    print(f"Result 3: {result3}")
```

### Questions

**1. Predict the output.** For each call:
- Call 1: Will it crash? If not, what will `result1` contain?
- Call 2: Will it crash? If not, what will `result2` contain?
- Call 3: Will it crash? If not, what will `result3` contain?

**2. Explain why.** A student says: "I added type hints, so Python will reject wrong types." Why is this belief incorrect? What actually happens when Python encounters type hints?

**3. Mental Trace:** Walk through what Python does when it sees `def process_user(user_id: int, username: str)`. At what point (if any) does Python check if the arguments match the declared types?

---

## Exercise 1.2: The Generic Illusion

```python
# generic_illusion.py
from typing import TypeVar

T = TypeVar("T")

def first(items: list[T]) -> T:
    """Return the first item from a list."""
    return items[0]

def identity(value: T) -> T:
    """Return the value unchanged."""
    return value

# --- What types does the type checker infer? ---

# Scenario A
numbers = [1, 2, 3]
result_a = first(numbers)
next_value = result_a + 10

# Scenario B
mixed = [1, "hello", 3.14]
result_b = first(mixed)

# Scenario C
empty: list[str] = []
# result_c = first(empty)  # Uncomment: what happens?

# Scenario D
x = identity(42)
y = identity("hello")
z = x + y  # What does the type checker say?
```

### Questions

**1. Predict the inferred types.** According to the type checker (not runtime):
- What is the type of `result_a`?
- What is the type of `result_b`? (Hint: what's `list[int | str | float]`'s element type?)
- If we uncomment `result_c = first(empty)`, what type would `result_c` be?
- What are the types of `x` and `y`?

**2. Explain the mechanism.** How does `TypeVar` "remember" what type was passed in? Is a new version of `first()` created for each call, or does something else happen?

**3. The trap in Scenario D:** Will `z = x + y` produce a type error? Explain why or why not, considering that `x` is `int` and `y` is `str`.

---

## Exercise 1.3: The Protocol Puzzle

```python
# protocol_puzzle.py
from typing import Protocol

class Flyer(Protocol):
    def fly(self) -> str:
        ...

# Note: Bird does NOT inherit from Flyer
class Bird:
    def fly(self) -> str:
        return "Flapping wings"
    
    def chirp(self) -> str:
        return "Tweet!"

# Note: Airplane does NOT inherit from Flyer
class Airplane:
    def fly(self) -> str:
        return "Jet engines roaring"
    
    def refuel(self) -> None:
        print("Refueling...")

# Note: Fish does NOT inherit from Flyer
class Fish:
    def swim(self) -> str:
        return "Swimming..."

def launch(flyer: Flyer) -> str:
    """Launch anything that can fly."""
    return f"Launching: {flyer.fly()}"

# --- Which calls will the type checker accept? ---
bird = Bird()
plane = Airplane()
fish = Fish()

result1 = launch(bird)
result2 = launch(plane)
result3 = launch(fish)
```

### Questions

**1. Predict the type checker's verdict.** For each call to `launch()`:
- `launch(bird)`: Accepted or rejected by type checker? Why?
- `launch(plane)`: Accepted or rejected by type checker? Why?
- `launch(fish)`: Accepted or rejected by type checker? Why?

**2. Explain the mechanism.** None of the classes inherit from `Flyer`. How does the type checker decide if they're compatible with the `Flyer` Protocol?

**3. Contrast with inheritance:** If `Flyer` were an abstract base class (ABC) instead of a Protocol, would `Bird` be accepted by `launch()`? What would need to change?

---

## Exercise 1.4: The Narrowing Necessity

```python
# narrowing_necessity.py
from typing import Union

def process_value(value: Union[str, int]) -> str:
    # Attempt 1: Direct method call
    if True:  # Always true, just for structure
        upper_result = value.upper()  # Line A
    
    # Attempt 2: Direct arithmetic
    doubled = value * 2  # Line B
    
    # Attempt 3: String formatting
    return f"Processed: {value}"  # Line C

def handle_optional(data: str | None) -> int:
    # Attempt: Direct method call
    return len(data)  # Line D
```

### Questions

**1. Predict which lines cause type errors.** For each marked line (A, B, C, D):
- Line A (`value.upper()`): Error or OK?
- Line B (`value * 2`): Error or OK?
- Line C (`f"Processed: {value}"`): Error or OK?
- Line D (`len(data)`): Error or OK?

**2. Explain the type checker's reasoning.** For each line that causes an error, explain:
- What type does the checker think the variable has?
- Why can't the checker allow the operation?
- What would happen at runtime if `value` were the "wrong" type for that operation?

**3. The fix (conceptual).** Without writing code, describe what you would need to add before Line A and Line D to make them type-safe.

---

# LEVEL 1: SOLUTION KEY

## Exercise 1.1 Solution

**Correct Output:**
```
Result 1: {'id': 'alice', 'name': 12345, 'display': 'User #alice: 12345'}
Result 2: {'id': [1, 2, 3], 'name': {'name': 'test'}, 'display': "User #[1, 2, 3]: {'name': 'test'}"}
Traceback (most recent call last):
  File "runtime_surprise.py", line 18, in <module>
    result3 = calculate_discount("fifty", "twenty")
  File "runtime_surprise.py", line 10, in calculate_discount
    return price * (1 - percentage / 100)
TypeError: unsupported operand type(s) for /: 'str' and 'int'
```

1. **The Why:**
- Calls 1 and 2 succeed because f-strings convert anything to string representation. Python never checks types.
- Call 3 crashes not because of type hints, but because `"twenty" / 100` is invalid—the crash is from the *operation*, not the annotation.

2. **Explain why:**
- The student's belief is incorrect because Python type hints (annotations) are ignored at runtime. When you write user_id: int, you are adding metadata to the function, but the Python interpreter does not enforce this. It treats the arguments as dynamic objects. If the object passed in supports the operations performed on it (like an f-string), the code runs. If it doesn't (like dividing a string), it crashes.

3. **Mental Trace:**
- When Python encounters def process_user(user_id: int, username: str):
  1. It compiles the function body.
  2. It evaluates the annotations (int and str) and stores them in the function's __annotations__ attribute.
  3. At no point does Python generate code to check arguments against these types.
  4. When called, Python simply assigns the first argument to user_id and the second to username, regardless of what they are.

**Key Insight:** Type hints are metadata Python stores but ignores. The crash in Call 3 would happen with or without type hints.

---

## Exercise 1.2 Solution

1. **Inferred Types:**
* result_a: int (Since numbers is list[int], T binds to int.)
* result_b: Union[int, str, float] (or object, depending on checker strictness). (Python infers the list is a mix of types, so the return value is a Union of those types.)
* result_c: str (Even though the list is empty and calling it would crash at runtime with an IndexError, the type checker trusts the definition list[str], so it believes the return value must be a str.)
* x and y: x is int, y is str.

2. **Explain the mechanism**
- TypeVar does not create new versions of the function (unlike C++ templates).
* Runtime: There is only one function first. It doesn't know or care about T.
* Static Analysis: The type checker treats T as a variable to solve for. When it sees first(numbers), it looks at numbers, sees list[int], matches that against list[T], solves T = int, and therefore concludes the return type T is int.

3. **The Trap in D:** Yes, `z = x + y` will produce a type error. Even though each variable has a specific type, `int + str` is invalid. The type checker correctly rejects this.

**Key Insight:** TypeVar doesn't create copies—it's a constraint that links input and output types during static analysis.

---

## Exercise 1.3 Solution

1. **Predict the type checker's verdict**
- `launch(bird)`: ✅ Accepted — Bird has a `fly() -> str` method
- `launch(plane)`: ✅ Accepted — Airplane has a `fly() -> str` method
- `launch(fish)`: ❌ Rejected — Fish has no `fly()` method

2. **Explain the mechanism**
- This is Structural Subtyping (often called static Duck Typing). Unlike nominal subtyping (inheritance), where you must explicitly say class Bird(Flyer), Protocols check if the class looks right. If the class has the required methods with the correct signatures, the type checker considers it compatible, even if the class author never heard of Flyer.

3. **If Flyer were an Abstract Base Class (ABC):**
* Bird would be Rejected.
* To fix it, Bird would have to explicitly inherit: class Bird(Flyer): ....
* Protocols allow for compatibility without modifying the original classes (which is useful if Bird comes from a 3rd party library you can't change).

**Key Insight:** Protocol uses structural subtyping—"does it have the shape?" not "does it inherit from the right class?"

---

## Exercise 1.4 Solution

1. **Predict which lines cause type errors**
- Line A: ❌ Error — `int` has no `.upper()` method
- Line B: ✅ OK — both `str` and `int` support `* 2`
- Line C: ✅ OK — f-strings work with any type
- Line D: ❌ Error — `len(None)` would crash

2. **Explain the type checker's reasoning**
* Line A: The checker knows value is str | int. It looks for .upper() on both. str has it, but int does not. Since value might be an integer at runtime, this is unsafe.
    * ```Runtime consequence: AttributeError: 'int' object has no attribute 'upper'.```
* Line D: The checker knows data is str | None. It looks for __len__. str has it, but None does not.
    * ```Runtime consequence: TypeError: object of type 'NoneType' has no len().```

3. **The fix (conceptual):**
You need Type Narrowing (or Type Guards). You must prove to the checker that the variable is the specific type required for the operation.
* Before Line A: Add a check: if isinstance(value, str):
    * Inside this if block, the checker knows value is definitely a str.
* Before Line D: Add a check: if data is not None:
    * Inside this if block, the checker knows data is definitely a str.

**Key Insight:** The type checker is conservative. It only allows operations valid for ALL types in the union.

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2: The Untyped Repository

### Context

A developer wrote a data repository but didn't leverage type hints properly. The code works but provides no type safety—the type checker can't help catch bugs.

```python
# untyped_repository.py
from typing import Any, Optional
from dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str
    email: str

@dataclass
class Product:
    id: int
    name: str
    price: float

class Repository:
    """A generic repository that stores any kind of entity."""
    
    def __init__(self) -> None:
        self._storage: dict[int, Any] = {}  # Line 1
    
    def add(self, item: Any) -> Any:  # Line 2
        self._storage[item.id] = item
        return item
    
    def get(self, id: int) -> Any:  # Line 3
        return self._storage.get(id)
    
    def get_all(self) -> list:  # Line 4
        return list(self._storage.values())
    
    def delete(self, id: int) -> bool:
        if id in self._storage:
            del self._storage[id]
            return True
        return False

# Usage that SHOULD be caught by type checker but isn't:
user_repo = Repository()
product_repo = Repository()

# Add a user
user_repo.add(User(id=1, name="Alice", email="alice@example.com"))

# BUG 1: Adding wrong type to user repository (Product instead of User)
user_repo.add(Product(id=2, name="Widget", price=9.99))  # Should error!

# BUG 2: Getting user but treating as Product
user = user_repo.get(1)
print(user.price)  # Should error! User has no 'price' attribute

# BUG 3: Mixed types in get_all
all_users = user_repo.get_all()
for u in all_users:
    print(u.email)  # Will crash on Product! But type checker allows it
```

---

## Task 2a: Identify the Flaw

This repository uses `Any` throughout, which disables type checking. Identify:

1. Which specific lines (marked 1-4) defeat the purpose of type hints?
2. For each line, what type safety is lost?

Do NOT fix the code yet. Just identify and explain.

---

## Task 2b: Explain the Mechanism

Explain what happens in the type checker's view:

1. When we write `user_repo.add(Product(...))`, why doesn't the type checker complain?
2. When we write `print(user.price)`, why doesn't the type checker complain?
3. What is `Any` actually telling the type checker to do?

---

## Task 2c: Prove the Problem

Create a test file that demonstrates the type-unsafety. Your test should:

1. Run successfully (no runtime errors in setup)
2. Show that the type checker (mypy) reports ZERO errors
3. Then show a runtime crash proving the type system didn't protect us

```python
# test_untyped.py

# TODO: Write code that:
# 1. Creates a Repository
# 2. Adds items of mixed types
# 3. Retrieves an item and calls a method that doesn't exist
# 4. Show mypy output (no errors) vs runtime output (crash)

# Include the mypy command and its output as a comment
```

---

## Task 2d: Fix the Code

Rewrite the Repository using generics to provide proper type safety:

Requirements:
- Use `TypeVar` with appropriate bounds
- Use `Generic[T]` for the class
- All methods should have proper type annotations
- The type checker should catch bugs 1, 2, and 3 from the original code

```python
# typed_repository.py
from typing import TypeVar, Generic, Optional
from dataclasses import dataclass

@dataclass
class BaseEntity:
    id: int

@dataclass
class User(BaseEntity):
    name: str
    email: str

@dataclass  
class Product(BaseEntity):
    name: str
    price: float

# TODO: Rewrite Repository with generics
# class Repository(Generic[???]):
#     ...
```

---

## Task 2e: Write Verification Test

Write a test that:
- FAILS type checking on the original `Repository` (actually, it PASSES because Any allows everything—that's the bug!)
- FAILS type checking on your generic `Repository` when misused (proving it catches bugs)
- PASSES type checking on your generic `Repository` when used correctly

```python
# test_typed_repository.py

def test_type_safety():
    """
    This test verifies that the generic Repository catches type errors
    that the original Any-based Repository missed.
    """
    # TODO: Write tests that demonstrate type safety
    # Show mypy output for both correct and incorrect usage
    pass
```

---

# LEVEL 2: SOLUTION KEY

## Task 2a — The Flaws

The flaw is the pervasive use of Any, which tells the type checker to ignore these values.
* Line 1: self._storage: dict[int, Any] = {}
    * Loss of Safety: The dictionary accepts mixed types. A repository intended for User objects can accidentally store Product objects, ints, or None, and the type checker won't notice.
* Line 2: def add(self, item: Any) -> Any:
    * Loss of Safety: The input argument is unchecked. We can pass a Product into a User repository. The return type is also Any, losing type information for the caller.
* Line 3: def get(self, id: int) -> Any:
    * Loss of Safety: The return value is Any. This disables attribute checking on the result. We can access non-existent attributes (like user.price) and the type checker will remain silent.
* Line 4: def get_all(self) -> list:
    * Loss of Safety: In Python, a bare list is treated as list[Any]. Iterating over this list yields items of type Any, propagating the lack of safety to any loop consuming this data.

---

## Task 2b — The Mechanism

1. **Why doesn't user_repo.add(Product(...)) complain?**
- The add method is defined as accepting item: Any. In the type system, Any is compatible with every other type. Therefore, passing a Product (or a str, or an int) satisfies the requirement of being Any.

2. **Why doesn't print(user.price) complain**
- The get method returns Any. When a variable is typed as Any, the type checker assumes the programmer knows better than the tool. It allows arbitrary attribute access and arbitrary method calls. The checker effectively says: "I don't know what this is, so I will allow you to do anything to it."

3. **What is Any actually telling the checker?**
- Any tells the type checker: "Silence all errors regarding this value. Assume valid operations will happen at runtime." It is an escape hatch from the type system.

---

## Task 2c — Proof of Problem

```python
# test_untyped.py
from untyped_repository import Repository, User, Product

# This code has zero type errors but crashes at runtime

user_repo = Repository()
user_repo.add(User(id=1, name="Alice", email="alice@example.com"))
user_repo.add(Product(id=2, name="Widget", price=9.99))  # Wrong type!

# Get what we think is a User
user = user_repo.get(2)  # Actually a Product!
print(user.email)  # Runtime crash: AttributeError: 'Product' has no 'email'

# mypy output: "Success: no issues found"
# Runtime output: AttributeError: 'Product' object has no attribute 'email'
```

---

## Task 2d — Fixed Code

```python
# typed_repository.py
from typing import TypeVar, Generic, Optional
from dataclasses import dataclass

@dataclass
class BaseEntity:
    id: int

@dataclass
class User(BaseEntity):
    name: str
    email: str

@dataclass
class Product(BaseEntity):
    name: str
    price: float

T = TypeVar("T", bound=BaseEntity)

class Repository(Generic[T]):
    """A generic repository with full type safety."""
    
    def __init__(self) -> None:
        self._storage: dict[int, T] = {}
    
    def add(self, item: T) -> T:
        self._storage[item.id] = item
        return item
    
    def get(self, id: int) -> Optional[T]:
        return self._storage.get(id)
    
    def get_all(self) -> list[T]:
        return list(self._storage.values())
    
    def delete(self, id: int) -> bool:
        if id in self._storage:
            del self._storage[id]
            return True
        return False

# Now with type safety:
user_repo: Repository[User] = Repository()
product_repo: Repository[Product] = Repository()

user_repo.add(User(id=1, name="Alice", email="alice@example.com"))
# user_repo.add(Product(id=2, name="Widget", price=9.99))  # ❌ Type error!

user = user_repo.get(1)
if user:
    print(user.email)  # ✅ Type checker knows user is User
    # print(user.price)  # ❌ Type error! User has no 'price'
```

---

## Task 2e — Verification Test

```python
# test_typed_repository.py
from typed_repository import Repository, User, Product

def test_correct_usage() -> None:
    """Type checker accepts correct usage."""
    repo: Repository[User] = Repository()
    repo.add(User(id=1, name="Alice", email="alice@example.com"))
    
    user = repo.get(1)
    if user:
        print(user.email)  # ✅ OK
    
    for u in repo.get_all():
        print(u.name)  # ✅ OK

def test_catches_wrong_type() -> None:
    """Type checker rejects adding wrong type."""
    repo: Repository[User] = Repository()
    
    # Uncomment to see type error:
    # repo.add(Product(id=1, name="Widget", price=9.99))
    # Error: Argument 1 to "add" has incompatible type "Product"; expected "User"

def test_catches_wrong_attribute() -> None:
    """Type checker rejects accessing wrong attribute."""
    repo: Repository[User] = Repository()
    repo.add(User(id=1, name="Alice", email="alice@example.com"))
    
    user = repo.get(1)
    if user:
        # Uncomment to see type error:
        # print(user.price)
        # Error: "User" has no attribute "price"
        pass

# Run: mypy test_typed_repository.py
# Expected: Errors on commented lines, success when properly used
```

---

# LEVEL 2.5: EXTEND EXISTING CODE

## Given Code

**Context:** Your teammate wrote a data validation system using Protocols. It's been working well for validating user input. The tests pass. Your task is to add features WITHOUT breaking existing behavior.

```python
# validator.py
from typing import Protocol, Any
from dataclasses import dataclass

class Validator(Protocol):
    """Protocol for any validation logic."""
    
    def validate(self, value: Any) -> bool:
        """Return True if value is valid, False otherwise."""
        ...
    
    def error_message(self) -> str:
        """Return the error message for invalid values."""
        ...

@dataclass
class ValidationResult:
    """Result of running validation."""
    is_valid: bool
    errors: list[str]

class NotEmptyValidator:
    """Validates that a string is not empty."""
    
    def validate(self, value: Any) -> bool:
        if not isinstance(value, str):
            return False
        return len(value.strip()) > 0
    
    def error_message(self) -> str:
        return "Value must not be empty"

class MinLengthValidator:
    """Validates minimum string length."""
    
    def __init__(self, min_length: int) -> None:
        self.min_length = min_length
    
    def validate(self, value: Any) -> bool:
        if not isinstance(value, str):
            return False
        return len(value) >= self.min_length
    
    def error_message(self) -> str:
        return f"Value must be at least {self.min_length} characters"

def run_validators(value: Any, validators: list[Validator]) -> ValidationResult:
    """Run all validators against a value."""
    errors: list[str] = []
    
    for validator in validators:
        if not validator.validate(value):
            errors.append(validator.error_message())
    
    return ValidationResult(
        is_valid=len(errors) == 0,
        errors=errors
    )
```

**Existing Tests (must continue to pass):**

```python
# test_validator.py
# run "pytest test_validator.py" to test
from validator import (
    NotEmptyValidator, MinLengthValidator, 
    run_validators, ValidationResult
)

def test_not_empty_passes() -> None:
    v = NotEmptyValidator()
    assert v.validate("hello") is True
    assert v.validate("  hello  ") is True

def test_not_empty_fails() -> None:
    v = NotEmptyValidator()
    assert v.validate("") is False
    assert v.validate("   ") is False
    assert v.validate(123) is False

def test_min_length_passes() -> None:
    v = MinLengthValidator(5)
    assert v.validate("hello") is True
    assert v.validate("hello world") is True

def test_min_length_fails() -> None:
    v = MinLengthValidator(5)
    assert v.validate("hi") is False
    assert v.validate("") is False

def test_run_validators_all_pass() -> None:
    validators = [NotEmptyValidator(), MinLengthValidator(3)]
    result = run_validators("hello", validators)
    assert result.is_valid is True
    assert result.errors == []

def test_run_validators_some_fail() -> None:
    validators = [NotEmptyValidator(), MinLengthValidator(10)]
    result = run_validators("hello", validators)
    assert result.is_valid is False
    assert "at least 10 characters" in result.errors[0]
```

---

## Extension A: Add a RegexValidator

**Requirement:** Create a `RegexValidator` class that validates strings against a regular expression pattern.

**Constraint:** Must match the `Validator` Protocol (no inheritance from it required).

**New test your code must pass:**

```python
import re

def test_regex_validator() -> None:
    email_pattern = r'^[\w\.-]+@[\w\.-]+\.\w+$'
    v = RegexValidator(email_pattern, "Must be valid email")
    
    assert v.validate("user@example.com") is True
    assert v.validate("invalid-email") is False
    assert v.validate(123) is False
    assert v.error_message() == "Must be valid email"

def test_regex_in_pipeline() -> None:
    validators = [
        NotEmptyValidator(),
        RegexValidator(r'^\d{3}-\d{4}$', "Must be phone format XXX-XXXX")
    ]
    
    assert run_validators("555-1234", validators).is_valid is True
    assert run_validators("5551234", validators).is_valid is False
```

All previous tests must still pass.

---

## Extension B: Add Conditional Validation

**Requirement:** Create a `ConditionalValidator` that only runs its inner validator when a condition function returns True.

**Constraint:** The condition function takes the value and returns bool.

**New test your code must pass:**

```python
def test_conditional_validator() -> None:
    # Only validate length if value is not empty
    conditional = ConditionalValidator(
        condition=lambda v: isinstance(v, str) and len(v) > 0,
        validator=MinLengthValidator(5),
        skip_message="Skipped (empty input)"
    )
    
    # Non-empty string too short → fails
    assert conditional.validate("hi") is False
    assert conditional.error_message() == "Value must be at least 5 characters"
    
    # Empty string → condition false → skip (returns True)
    assert conditional.validate("") is True
    
    # Valid string → passes
    assert conditional.validate("hello") is True

def test_conditional_in_pipeline() -> None:
    validators = [
        ConditionalValidator(
            condition=lambda v: v is not None,
            validator=NotEmptyValidator(),
            skip_message="Skipped (None)"
        )
    ]
    
    assert run_validators(None, validators).is_valid is True  # Skipped
    assert run_validators("", validators).is_valid is False   # Not skipped, fails
```

All previous tests must still pass.

---

## Extension C: Add Generic Type-Safe Validators

**Requirement:** Create a generic `TypedValidator[T]` Protocol and a `RangeValidator` for numeric types that's type-safe.

**Constraint:** The type checker should catch attempts to use `RangeValidator` with non-numeric types.

**New test your code must pass:**

```python
def test_range_validator() -> None:
    v: RangeValidator[int] = RangeValidator(min_val=1, max_val=100)
    
    assert v.validate(50) is True
    assert v.validate(0) is False
    assert v.validate(101) is False
    assert "between 1 and 100" in v.error_message()

def test_range_validator_float() -> None:
    v: RangeValidator[float] = RangeValidator(min_val=0.0, max_val=1.0)
    
    assert v.validate(0.5) is True
    assert v.validate(1.5) is False

def test_typed_validator_protocol() -> None:
    """Type checker should catch this if uncommented:
    
    v: RangeValidator[str] = RangeValidator(min_val="a", max_val="z")
    # Error: str doesn't support < and > comparison in a meaningful numeric way
    # (Actually str does support comparison, so this test is conceptual)
    """
    # This extension is primarily about demonstrating generic protocols
    pass
```

All previous tests must still pass.

---

## Regression Contract

After completing all extensions, the following must be true:

1. All 6 original tests pass unchanged
2. All new extension tests pass
3. `NotEmptyValidator`, `MinLengthValidator`, and `run_validators` work identically
4. New validators conform to the `Validator` Protocol
5. Type checker (mypy) reports no errors

---

# LEVEL 2.5: SOLUTION KEY

## Extension A Solution

```python
import re
from typing import Any

class RegexValidator:
    """Validates strings against a regex pattern."""
    
    def __init__(self, pattern: str, error_msg: str) -> None:
        self.pattern = re.compile(pattern)
        self._error_msg = error_msg
    
    def validate(self, value: Any) -> bool:
        if not isinstance(value, str):
            return False
        return self.pattern.match(value) is not None
    
    def error_message(self) -> str:
        return self._error_msg
```

---

## Extension B Solution

```python
from typing import Any, Callable

class ConditionalValidator:
    """Validator that only runs when condition is met."""
    
    def __init__(
        self, 
        condition: Callable[[Any], bool],
        validator: Validator,
        skip_message: str
    ) -> None:
        self.condition = condition
        self.validator = validator
        self.skip_message = skip_message
        self._last_skipped = False
    
    def validate(self, value: Any) -> bool:
        if not self.condition(value):
            self._last_skipped = True
            return True  # Skip validation, consider valid
        
        self._last_skipped = False
        return self.validator.validate(value)
    
    def error_message(self) -> str:
        if self._last_skipped:
            return self.skip_message
        return self.validator.error_message()
```

---

## Extension C Solution

```python
from typing import TypeVar, Generic, Protocol

# Define a comparable type for numeric values
Numeric = TypeVar("Numeric", int, float)

class TypedValidator(Protocol[Numeric]):
    """Generic protocol for type-safe validators."""
    
    def validate(self, value: Numeric) -> bool:
        ...
    
    def error_message(self) -> str:
        ...

class RangeValidator(Generic[Numeric]):
    """Validates that a numeric value is within a range."""
    
    def __init__(self, min_val: Numeric, max_val: Numeric) -> None:
        self.min_val = min_val
        self.max_val = max_val
    
    def validate(self, value: Any) -> bool:
        if not isinstance(value, (int, float)):
            return False
        return self.min_val <= value <= self.max_val
    
    def error_message(self) -> str:
        return f"Value must be between {self.min_val} and {self.max_val}"
```

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## Scenario

You're building a **plugin system** for a text processing application. Different plugins can transform text in various ways (uppercase, reverse, add prefix, etc.). The system must be type-safe and extensible.

**Input:** Raw text and a list of plugins to apply

**Output:** Transformed text after all plugins run

**Constraints:**

| Constraint | Specification | Why It Matters |
|------------|---------------|----------------|
| Type Safety | All plugins must conform to a Protocol | Catch incompatible plugins at type-check time |
| Composability | Plugins can be chained in any order | Flexible transformation pipelines |
| Extensibility | New plugins can be added without modifying core code | Open/closed principle |
| Generics | Support plugins that work on different types (str, bytes, etc.) | Reusable across data types |

---

## Starter Code

```python
# plugin_system.py
from typing import Protocol, TypeVar, Generic, Callable
from dataclasses import dataclass

# Type variable for the data being processed
T = TypeVar("T")

# TODO Phase 3a: Define the Plugin Protocol
# class Plugin(Protocol[T]):
#     ...

# TODO Phase 3b: Implement the Pipeline class
# class Pipeline(Generic[T]):
#     ...

# TODO Phase 3c: Implement these concrete plugins
# class UppercasePlugin: ...
# class ReversePlugin: ...
# class PrefixPlugin: ...

# Helper for testing
def create_text_pipeline() -> "Pipeline[str]":
    """Create a pipeline for text processing."""
    # TODO: Return a configured Pipeline
    pass

# Test harness (do not modify)
def main() -> None:
    # Test 1: Basic pipeline
    pipeline = create_text_pipeline()
    pipeline.add_plugin(UppercasePlugin())
    pipeline.add_plugin(ReversePlugin())
    
    result = pipeline.process("hello")
    assert result == "OLLEH", f"Expected OLLEH, got {result}"
    print("✅ Test 1 passed: Basic pipeline works")
    
    # Test 2: Plugin ordering matters
    pipeline2: Pipeline[str] = Pipeline()
    pipeline2.add_plugin(ReversePlugin())
    pipeline2.add_plugin(UppercasePlugin())
    
    result2 = pipeline2.process("hello")
    assert result2 == "OLLEH", f"Expected OLLEH, got {result2}"
    print("✅ Test 2 passed: Order doesn't affect final result for these plugins")
    
    # Test 3: Configurable plugin
    pipeline3: Pipeline[str] = Pipeline()
    pipeline3.add_plugin(PrefixPlugin(">>> "))
    pipeline3.add_plugin(UppercasePlugin())
    
    result3 = pipeline3.process("hello")
    assert result3 == ">>> HELLO", f"Expected '>>> HELLO', got '{result3}'"
    print("✅ Test 3 passed: Configurable plugins work")
    
    # Test 4: Type safety (this should be caught by type checker)
    # Uncomment to verify type error:
    # number_pipeline: Pipeline[int] = Pipeline()
    # number_pipeline.add_plugin(UppercasePlugin())  # Should error: UppercasePlugin is for str
    
    print("\n✅ All tests passed!")

if __name__ == "__main__":
    main()
```

---

## Phase 3a: Define the Protocol

Implement the `Plugin` Protocol:

Requirements:
- Generic over type `T` (the data type it processes)
- Has a `transform(self, data: T) -> T` method
- Has a `name` property that returns `str`

Verification: Type definition compiles without errors.

---

## Phase 3b: Implement the Pipeline

Implement the `Pipeline` class:

Requirements:
- Generic over type `T`
- `add_plugin(plugin: Plugin[T]) -> None` — adds a plugin to the pipeline
- `process(data: T) -> T` — runs data through all plugins in order
- `list_plugins() -> list[str]` — returns names of all plugins

Verification: Can create a `Pipeline[str]` and add string plugins to it.

---

## Phase 3c: Implement Concrete Plugins

Implement three plugins:

1. **UppercasePlugin**: Transforms string to uppercase
2. **ReversePlugin**: Reverses the string  
3. **PrefixPlugin**: Adds a configurable prefix to the string

Requirements:
- Each must satisfy the `Plugin[str]` Protocol
- Each must have a descriptive `name` property
- `PrefixPlugin` must accept the prefix in its constructor

Verification: All assertions in `main()` pass.

---

## Phase 3d: Write Your Own Test

Write ONE additional test that verifies an edge case:

```python
def test_edge_case() -> None:
    """
    Edge case: [Your description]
    Why it matters: [Why this could break in production]
    """
    # Your test implementation
    pass
```

Examples of good edge cases:
- Empty pipeline (no plugins)
- Empty string input
- Plugin that returns empty string
- Very long plugin chain

---

# LEVEL 3: SOLUTION KEY

## Phase 3a — Protocol Definition

```python
from typing import Protocol, TypeVar

T = TypeVar("T")

class Plugin(Protocol[T]):
    """Protocol for transformation plugins."""
    
    @property
    def name(self) -> str:
        """Return the plugin's display name."""
        ...
    
    def transform(self, data: T) -> T:
        """Transform the input data and return the result."""
        ...
```

---

## Phase 3b — Pipeline Implementation

```python
from typing import Generic, TypeVar

T = TypeVar("T")

class Pipeline(Generic[T]):
    """A configurable chain of plugins."""
    
    def __init__(self) -> None:
        self._plugins: list[Plugin[T]] = []
    
    def add_plugin(self, plugin: Plugin[T]) -> None:
        """Add a plugin to the end of the pipeline."""
        self._plugins.append(plugin)
    
    def process(self, data: T) -> T:
        """Run data through all plugins in order."""
        result = data
        for plugin in self._plugins:
            result = plugin.transform(result)
        return result
    
    def list_plugins(self) -> list[str]:
        """Return names of all plugins in order."""
        return [p.name for p in self._plugins]
```

---

## Phase 3c — Concrete Plugins

```python
class UppercasePlugin:
    """Converts text to uppercase."""
    
    @property
    def name(self) -> str:
        return "Uppercase"
    
    def transform(self, data: str) -> str:
        return data.upper()

class ReversePlugin:
    """Reverses the text."""
    
    @property
    def name(self) -> str:
        return "Reverse"
    
    def transform(self, data: str) -> str:
        return data[::-1]

class PrefixPlugin:
    """Adds a prefix to the text."""
    
    def __init__(self, prefix: str) -> None:
        self._prefix = prefix
    
    @property
    def name(self) -> str:
        return f"Prefix({self._prefix!r})"
    
    def transform(self, data: str) -> str:
        return self._prefix + data
```

---

## Phase 3d — Example Edge Case Tests

```python
def test_empty_pipeline() -> None:
    """
    Edge case: Pipeline with no plugins
    Why it matters: Should return input unchanged, not crash
    """
    pipeline: Pipeline[str] = Pipeline()
    result = pipeline.process("hello")
    assert result == "hello", "Empty pipeline should return input unchanged"

def test_empty_string_input() -> None:
    """
    Edge case: Empty string through plugins
    Why it matters: Plugins might assume non-empty input
    """
    pipeline: Pipeline[str] = Pipeline()
    pipeline.add_plugin(UppercasePlugin())
    pipeline.add_plugin(PrefixPlugin(">>> "))
    
    result = pipeline.process("")
    assert result == ">>> ", "Should handle empty string gracefully"

def test_plugin_returns_empty() -> None:
    """
    Edge case: Plugin that returns empty string
    Why it matters: Subsequent plugins should handle empty input
    """
    class ClearPlugin:
        @property
        def name(self) -> str:
            return "Clear"
        
        def transform(self, data: str) -> str:
            return ""
    
    pipeline: Pipeline[str] = Pipeline()
    pipeline.add_plugin(ClearPlugin())
    pipeline.add_plugin(PrefixPlugin(">>> "))
    
    result = pipeline.process("hello")
    assert result == ">>> ", "Should handle plugin returning empty string"
```

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## Part A: The Over-Engineering Trap

A developer heard "always use generics for reusable code" and wrote this:

```python
# over_engineered.py
from typing import TypeVar, Generic, Protocol, Callable, Optional
from dataclasses import dataclass

T = TypeVar("T")
R = TypeVar("R")
E = TypeVar("E", bound=Exception)

class Transformer(Protocol[T, R]):
    def transform(self, value: T) -> R:
        ...

class ResultHandler(Protocol[R, E]):
    def handle_success(self, result: R) -> None:
        ...
    def handle_error(self, error: E) -> None:
        ...

@dataclass
class GenericProcessor(Generic[T, R, E]):
    transformer: Transformer[T, R]
    handler: ResultHandler[R, E]
    
    def process(self, value: T) -> Optional[R]:
        try:
            result = self.transformer.transform(value)
            self.handler.handle_success(result)
            return result
        except Exception as e:
            self.handler.handle_error(e)  # type: ignore
            return None

# Implementation for a simple task: double a number
class NumberDoubler:
    def transform(self, value: int) -> int:
        return value * 2

class PrintHandler:
    def handle_success(self, result: int) -> None:
        print(f"Result: {result}")
    
    def handle_error(self, error: Exception) -> None:
        print(f"Error: {error}")

# Usage
processor: GenericProcessor[int, int, Exception] = GenericProcessor(
    transformer=NumberDoubler(),
    handler=PrintHandler()
)
result = processor.process(5)
print(f"Final: {result}")
```

### Questions

**1. What is the actual task this code accomplishes?** Describe in one sentence.

**2. Write the simple version.** Implement the same functionality without generics, protocols, or the processor pattern. How many lines does it take?

**3. Compare the costs:**
   - Lines of code: Generic version vs Simple version
   - Concepts a new developer must understand: List them for each version
   - What bugs could the generic version catch that the simple version wouldn't?

**4. When WOULD this pattern be justified?** Describe a scenario where this level of abstraction pays off.

---

## Part B: The Type Hint Overhead

```python
# overhead_comparison.py
from typing import TypedDict, Protocol, TypeVar, Generic

# Version A: Full type annotations
class UserDict(TypedDict):
    id: int
    name: str
    email: str

T = TypeVar("T", bound=UserDict)

class Repository(Protocol[T]):
    def get(self, id: int) -> T | None: ...
    def save(self, item: T) -> T: ...

class UserRepository:
    def __init__(self) -> None:
        self._data: dict[int, UserDict] = {}
    
    def get(self, id: int) -> UserDict | None:
        return self._data.get(id)
    
    def save(self, item: UserDict) -> UserDict:
        self._data[item["id"]] = item
        return item

def get_user_email(repo: Repository[UserDict], user_id: int) -> str | None:
    user = repo.get(user_id)
    if user is not None:
        return user["email"]
    return None


# Version B: Minimal annotations
class SimpleRepo:
    def __init__(self):
        self._data = {}
    
    def get(self, id):
        return self._data.get(id)
    
    def save(self, item):
        self._data[item["id"]] = item
        return item

def simple_get_email(repo, user_id):
    user = repo.get(user_id)
    if user:
        return user.get("email")
    return None
```

### Questions

**5. For a script that runs once and processes 10 users, which version is more appropriate? Why?**

**6. At what scale (team size, codebase size, project duration) do the type hints "pay off"?**

**7. Create a decision rule:** Complete this framework:

```
Use comprehensive type hints when:
- [ ] _______________
- [ ] _______________
- [ ] _______________

Use minimal/no type hints when:
- [ ] _______________
- [ ] _______________
```

---

## Part C: The Protocol Misuse

```python
# protocol_misuse.py
from typing import Protocol

class Addable(Protocol):
    def __add__(self, other: "Addable") -> "Addable":
        ...

def sum_all(items: list[Addable]) -> Addable:
    """Sum all items in the list."""
    result = items[0]
    for item in items[1:]:
        result = result + item
    return result

# This works:
numbers = [1, 2, 3]
total = sum_all(numbers)  # 6

# But so does this:
strings = ["a", "b", "c"]
combined = sum_all(strings)  # "abc"

# And this:
mixed = [1, "hello"]  # type: ignore
weird = sum_all(mixed)  # TypeError at runtime!
```

**8. What's wrong with this Protocol definition?** Why doesn't it prevent the mixed-type list?

**9. Write a better solution** that either:
   - Uses generics to ensure all items are the same type, OR
   - Uses `@overload` to provide specific signatures for known types

---

# LEVEL 4: SOLUTION KEY

## Part A Solutions

**1. Actual task:** Double a number and print the result.

**2. Simple version:**
```python
def double_and_print(value: int) -> int:
    result = value * 2
    print(f"Result: {result}")
    return result

double_and_print(5)
```
**3 lines** vs **35+ lines** for the generic version.

**3. Cost comparison:**

| Metric | Generic Version | Simple Version |
|--------|-----------------|----------------|
| Lines of code | 35+ | 3 |
| Concepts required | Generics, Protocols, TypeVar, dataclass, Optional | Functions |
| Bugs caught | None additional for this use case | N/A |

**4. When justified:** The generic pattern makes sense when:
- You have 5+ different transformers with consistent processing logic
- Error handling must be uniform across all transformers
- You're building a framework others will extend

---

## Part B Solutions

**5. For a 10-user script:** Version B (minimal annotations) is more appropriate. The overhead of defining TypedDict, Protocol, and generic Repository provides no benefit for a short-lived script.

**6. Type hints pay off when:**
- Team size > 2 developers
- Codebase > 1000 lines
- Project duration > 3 months
- Code will be modified by people who didn't write it

**7. Decision rule:**

```
Use comprehensive type hints when:
- [x] Multiple developers work on the codebase
- [x] Code will be maintained for months/years
- [x] Public API that others consume
- [x] Complex data structures passed between functions

Use minimal/no type hints when:
- [x] One-off scripts or experiments
- [x] Prototype code that will be rewritten
- [x] Simple scripts under 100 lines with obvious types
```

---

## Part C Solutions

**8. Protocol flaw:** The `Addable` Protocol doesn't constrain what type you're adding. `int.__add__` and `str.__add__` both match, but they can't be added together. The Protocol checks structural compatibility but not type relationships.

**9. Better solution with generics:**

```python
from typing import TypeVar

T = TypeVar("T", int, float, str)  # Constrained to known addable types

def sum_all(items: list[T]) -> T:
    """Sum all items - all must be same type."""
    result = items[0]
    for item in items[1:]:
        result = result + item  # type: ignore (known limitation)
    return result

# Now the type checker enforces same-type lists:
numbers: list[int] = [1, 2, 3]
total = sum_all(numbers)  # OK, returns int

strings: list[str] = ["a", "b"]
combined = sum_all(strings)  # OK, returns str

# This won't type-check:
# mixed: list[int | str] = [1, "hello"]
# sum_all(mixed)  # Error: list[int | str] doesn't match list[T]
```

---

# LEVEL 5: THE TEACH-BACK

## Scenario

You're reviewing a pull request from a junior developer who's learning type hints. They've left comments explaining their understanding:

```python
# pr_submission.py
"""
User management module with type hints.

PR Comment from Junior:
"I added type hints everywhere! Python will now check types at runtime
and reject invalid data. This makes our code much safer."
"""

from typing import Any, List, Dict, Optional, Union

# Junior's comment: "I used Any here because the repository could store
# anything - users, products, etc. This makes it super flexible!"
class Repository:
    def __init__(self) -> None:
        self._data: Dict[int, Any] = {}
    
    def save(self, id: int, item: Any) -> None:
        self._data[id] = item
    
    def get(self, id: int) -> Any:
        return self._data.get(id)

# Junior's comment: "I made this function handle both string and int IDs
# because our frontend sometimes sends strings. Union types handle both
# automatically!"
def fetch_user(user_id: Union[str, int]) -> Dict:
    if user_id == "":
        return {}
    
    # Junior's comment: "Since it's Union[str, int], I can just convert
    # directly - Python knows what type it is"
    numeric_id = int(user_id)  # Convert to int for database
    
    repo = Repository()
    return repo.get(numeric_id) or {}

# Junior's comment: "I created a Protocol because I heard they're better
# than ABCs. Now any class is automatically a Printable!"
from typing import Protocol

class Printable(Protocol):
    def __str__(self) -> str:
        ...

def log_item(item: Printable) -> None:
    # Junior's comment: "Since everything in Python has __str__, every
    # object matches Printable. So this accepts anything safely!"
    print(f"Logging: {item}")

# Junior's comment: "Generic functions are cool! T means 'any type' so
# this function works with everything."
from typing import TypeVar

T = TypeVar("T")

def first_or_none(items: List[T]) -> Optional[T]:
    # Junior's comment: "Optional means it might be None, so I need to
    # check before returning. But I also check for empty list!"
    if items is None:
        return None
    if len(items) == 0:
        return None
    return items[0]

# Junior's comment: "I know this works because I tested it!"
if __name__ == "__main__":
    # Test the repository
    repo = Repository()
    repo.save(1, {"name": "Alice"})
    repo.save(2, "just a string")  # I can store anything!
    
    # Test fetch_user
    user = fetch_user("123")
    print(user)
    
    # Test log_item
    log_item("hello")
    log_item(42)
    log_item(None)  # This works too!
```

---

## Your Task: Write a Code Review Response

Your response must:

**1. Correct each misconception** specifically. Quote their exact comment and provide the correction.

**2. Explain the actual mechanism** for at least two key misconceptions.

**3. Identify any bugs** in the code. Explain what could go wrong and how to fix it.

**4. Provide a mental model** — give them a metaphor or framework they can use going forward.

**5. Connect to prior knowledge** — reference how generics, protocols, and type narrowing relate to each other.

**Tone Requirements:**
- Be respectful and encouraging
- Don't mock their attempt — they're learning
- Frame corrections as clarifications, not criticisms
- Include at least one "I had this confusion too" moment

---

# LEVEL 5: SOLUTION KEY (Model Review)

---

## Code Review: Type Hints PR

Hi! Thanks for adding type hints — this is great to see. You've clearly put effort into understanding the concepts. I want to clarify a few things that tripped me up when I was learning too.

---

### Misconception 1: Runtime Enforcement

> "Python will now check types at runtime and reject invalid data"

**Clarification:** Python completely ignores type hints when running code. They're metadata that external tools (like mypy, pyright, or your IDE) analyze *before* you run the code.

Think of type hints like a spell-checker for a document: Word doesn't prevent you from printing a document with spelling errors — it just shows red squiggles to warn you. Similarly, Python doesn't prevent you from running code with type mismatches; the type checker just warns you.

**What this means:** You need to run `mypy your_file.py` or use an IDE that runs a type checker. The hints alone do nothing.

---

### Misconception 2: Any = Flexibility

> "I used Any here because the repository could store anything"

**Clarification:** `Any` doesn't mean "flexible" — it means "I'm disabling type checking for this value." It's an escape hatch, not a feature.

When you use `Any`, the type checker stops helping you. Watch what happens:

```python
repo = Repository()
repo.save(1, {"name": "Alice"})
user = repo.get(1)  # type: Any
print(user.email)   # No type error! But crashes at runtime.
```

**The fix:** Use generics! I see you understand `TypeVar` — use that here:

```python
T = TypeVar("T")

class Repository(Generic[T]):
    def __init__(self) -> None:
        self._data: dict[int, T] = {}
    
    def get(self, id: int) -> T | None:
        return self._data.get(id)

# Now type-safe:
user_repo: Repository[User] = Repository()
```

---

### Misconception 3: Union Auto-Handles Both Types

> "Union types handle both automatically!"

**Clarification:** `Union[str, int]` tells the type checker "this could be either," but it doesn't automatically convert or dispatch. The type checker is *conservative* — it only allows operations that work for *both* types.

Your code has a bug here:

```python
numeric_id = int(user_id)  # What if user_id is already int?
```

Actually, this works (int(5) returns 5), but the type checker doesn't know that `user_id` is specifically `str` or `int` at this line. You need **type narrowing**:

```python
def fetch_user(user_id: str | int) -> dict:
    if isinstance(user_id, str):
        # Type checker knows: user_id is str here
        numeric_id = int(user_id)
    else:
        # Type checker knows: user_id is int here
        numeric_id = user_id
    
    return repo.get(numeric_id) or {}
```

---

### Misconception 4: Protocol Matches Everything

> "Since everything in Python has `__str__`, every object matches Printable"

**Clarification:** You're actually right that everything has `__str__`! But this means your Protocol is too broad to be useful. A Protocol that matches everything provides no type safety.

Protocols are valuable when they define a *specific* interface that not everything has:

```python
class Serializable(Protocol):
    def to_json(self) -> str:
        ...

# Now only classes with to_json() match
```

Your `log_item(item: Printable)` is equivalent to `log_item(item: object)` — which is fine, but the Protocol adds complexity without adding safety.

---

### Misconception 5: TypeVar Means "Any Type"

> "T means 'any type' so this function works with everything"

**Clarification:** This one's subtle. `T` doesn't mean "any type" — it means "some specific type that's the same everywhere T appears."

Your `first_or_none` function is actually correct! The beauty is:

```python
names = ["Alice", "Bob"]
first_name = first_or_none(names)  # first_name: str | None

numbers = [1, 2, 3]
first_num = first_or_none(numbers)  # first_num: int | None
```

The `T` creates a *relationship*: "whatever goes in, that same type comes out." This is different from `Any`, which breaks all type relationships.

---

### Bug Found: Null Check Order

```python
def first_or_none(items: List[T]) -> Optional[T]:
    if items is None:  # This check is suspicious
```

If `items` is typed as `List[T]` (not `Optional[List[T]]`), then `items` can never be `None` according to the type signature. Either:

1. The signature lies and should be `items: list[T] | None`, OR
2. The `if items is None` check is dead code

Pick one and make the code match the types.

---

### Mental Model: Contracts, Not Enforcement

Here's how I think about it:

> **Type hints are contracts written for the type checker, not for Python.**

When you write `def greet(name: str)`, you're not telling Python to check. You're promising: "I, the developer, guarantee this will receive a string. Type checker, please hold me to that promise everywhere I call this function."

The three concepts connect like this:

- **Generics (TypeVar):** "Same type in, same type out" — preserves type relationships
- **Protocols:** "Has these methods" — defines capabilities without requiring inheritance
- **Type Narrowing:** "In this branch, I know it's specifically X" — resolves union ambiguity

---

### Summary of Changes Needed

1. Replace `Any` with `Generic[T]` in Repository
2. Add type narrowing (`isinstance`) before using union-typed values
3. Consider whether `Printable` Protocol adds value (it might not)
4. Fix the `List[T]` vs `Optional[List[T]]` inconsistency
5. Run `mypy` on your code — you'll catch more issues!

Keep going — you're asking the right questions. Type hints clicked for me after I made these same mistakes. 🎉

---

# END OF EXERCISE SET