## LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PAIN FIRST, SOLUTION SECOND                                    │
│  ────────────────────────────                                   │
│  Students must FEEL the chaos of manual validation              │
│  before Pydantic feels like a relief, not a burden.             │
│                                                                 │
│  BRIDGE FROM WHAT THEY KNOW                                     │
│  ──────────────────────────                                     │
│  Dataclasses (Week 1) → BaseModel evolution.                    │
│  Type hints (Week 1) → They ARE the schema.                     │
│  Decorators (Week 1) → They ARE the validators.                 │
│  FastAPI (This week) → Pydantic IS the request/response layer.  │
│                                                                 │
│  ANALOGY-GROUNDED                                               │
│  ────────────────                                               │
│  Pydantic is a GATEKEEPER. Every concept maps to a              │
│  checkpoint that data must pass through to enter your API.      │
│                                                                 │
│  BUILD INTUITION THROUGH BREAKAGE                               │
│  ─────────────────────────────────                              │
│  Show what HAPPENS when bad data gets through.                  │
│  Then show Pydantic catching it. Every. Single. Time.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                    PYDANTIC DEEP DIVE                           │
│                     (3.5-4 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (30 min)                                   │
│  ├─ 1.1 The Manual Validation Nightmare (Demonstration)         │
│  ├─ 1.2 Why Dataclasses Aren't Enough                           │
│  ├─ 1.3 Enter Pydantic                                          │
│  └─ 1.4 The Gatekeeper Mental Model                             │
│                                                                 │
│  PART 2: BASEMODEL FUNDAMENTALS (50 min)                        │
│  ├─ 2.1 Your First Model (Schema as Class)                      │
│  ├─ 2.2 Type Coercion (Smart Parsing)                           │
│  ├─ 2.3 Validation Errors (Rejecting Bad Data)                  │
│  ├─ 2.4 Field Constraints (The Precision Rules)                 │
│  └─ 2.5 Optional Fields and Defaults                            │
│                                                                 │
│  PART 3: NESTED MODELS (30 min)                                 │
│  ├─ 3.1 Models Inside Models (Composition)                      │
│  ├─ 3.2 Collections of Models                                   │
│  └─ 3.3 Deep Validation (Errors Point to the Exact Spot)        │
│                                                                 │
│  PART 4: CUSTOM VALIDATORS (45 min)                             │
│  ├─ 4.1 @field_validator — Single Field Logic                   │
│  ├─ 4.2 Validation Modes (before vs after)                      │
│  ├─ 4.3 @model_validator — Cross-Field Logic                    │
│  └─ 4.4 Common Mistakes with Validators                         │
│                                                                 │
│  PART 5: CONFIGURATION & SERIALIZATION (40 min)                 │
│  ├─ 5.1 Model Configuration (model_config)                      │
│  ├─ 5.2 from_attributes (Reading from Objects)                  │
│  ├─ 5.3 Aliases (External vs Internal Names)                    │
│  └─ 5.4 Serialization Control (.model_dump())                   │
│                                                                 │
│  PART 6: REQUEST VS RESPONSE MODELS (30 min)                    │
│  ├─ 6.1 Why One Model Isn't Enough                              │
│  ├─ 6.2 The Create / Read / Update Pattern                      │
│  ├─ 6.3 Inheritance for Shared Fields                           │
│  └─ 6.4 Putting It Together with FastAPI                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Manual Validation Nightmare

**Start with a demonstration. Make them feel the tedium.**

```python
# demo_manual.py — A FastAPI endpoint with MANUAL validation
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.post("/products")
async def create_product(data: dict):
    """Create a product. Validate EVERYTHING by hand."""
    
    # --- Check required fields exist ---
    if "name" not in data:
        raise HTTPException(400, "Missing field: name")
    if "price" not in data:
        raise HTTPException(400, "Missing field: price")
    if "category" not in data:
        raise HTTPException(400, "Missing field: category")
    
    # --- Check types ---
    if not isinstance(data["name"], str):
        raise HTTPException(400, "name must be a string")
    if not isinstance(data["price"], (int, float)):
        raise HTTPException(400, "price must be a number")
    if not isinstance(data["category"], str):
        raise HTTPException(400, "category must be a string")
    
    # --- Check constraints ---
    if len(data["name"]) < 1:
        raise HTTPException(400, "name cannot be empty")
    if len(data["name"]) > 100:
        raise HTTPException(400, "name is too long (max 100)")
    if data["price"] <= 0:
        raise HTTPException(400, "price must be positive")
    if data["category"] not in ["electronics", "clothing", "food"]:
        raise HTTPException(400, "invalid category")
    
    # --- Handle optional fields with defaults ---
    description = data.get("description", "")
    if not isinstance(description, str):
        raise HTTPException(400, "description must be a string")
    
    stock = data.get("stock", 0)
    if not isinstance(stock, int):
        raise HTTPException(400, "stock must be an integer")
    if stock < 0:
        raise HTTPException(400, "stock cannot be negative")
    
    # After ALL that... finally create it
    product = {
        "name": data["name"],
        "price": data["price"],
        "category": data["category"],
        "description": description,
        "stock": stock,
    }
    return {"status": "created", "product": product}
```

**Count the lines. 40+ lines of validation for 5 fields.**

**Now ask the class:**

> "This is 5 fields. A real product model might have 20-30 fields. How many lines of validation is that? What happens when the requirements change — say, price now needs a maximum? How many places do you have to edit?"

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE PROBLEMS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. VOLUME        — 40 lines for 5 fields. 200 for 25?         │
│  2. FRAGILITY     — Miss one check? Bug in production.          │
│  3. INCONSISTENCY — Error messages differ across endpoints.     │
│  4. NO REUSE      — Copy-paste validation into every endpoint.  │
│  5. ONE-AT-A-TIME — First error stops. Client fixes one,        │
│                      submits, hits the NEXT error. Repeat.      │
│  6. NO DOCS       — Swagger UI has no idea what's valid.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.2 Why Dataclasses Aren't Enough

**Connection to what you've learned:**

> "In Week 1, you learned dataclasses. They give structure. Let's try using them."

```python
from dataclasses import dataclass

@dataclass
class Product:
    name: str
    price: float
    stock: int = 0
```

**Looks clean. But watch this:**

```python
# Create a product with COMPLETELY WRONG data
product = Product(name=99999, price="not a number", stock=-50)

print(product.name)   # 99999       ← An int in a str field!
print(product.price)  # not a number ← A string in a float field!
print(product.stock)  # -50          ← Negative stock!

print(type(product.name))   # <class 'int'>
print(type(product.price))  # <class 'str'>
```

**No errors. No warnings. It just... stores the wrong types.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  DATACLASS vs WHAT WE NEED                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DATACLASS:                                                     │
│  ──────────                                                     │
│  ✅ Structured storage     (fields with names and types)        │
│  ✅ __init__ generated     (no boilerplate constructor)         │
│  ✅ __repr__ generated     (readable printing)                  │
│  ❌ NO type checking       (types are just documentation)       │
│  ❌ NO validation          (any value accepted)                 │
│  ❌ NO coercion            ("5" stays "5", not int 5)           │
│  ❌ NO constraints         (negative stock? Fine.)              │
│                                                                 │
│  WHAT WE NEED:                                                  │
│  ─────────────                                                  │
│  Everything dataclass does, PLUS:                               │
│  ✅ Type enforcement       (wrong type → error)                 │
│  ✅ Automatic validation   (constraints checked on creation)    │
│  ✅ Smart coercion         ("5" → int 5 when sensible)          │
│  ✅ Rich error messages    (what went wrong, where, why)        │
│  ✅ Serialization          (to dict, to JSON, control output)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight:**

> "Dataclasses are filing cabinets — they store whatever you put in, no questions asked. We need a gatekeeper who CHECKS everything at the door."

---

## 1.3 Enter Pydantic

**Same model. Different behavior.**

```python
from pydantic import BaseModel

class Product(BaseModel):
    name: str
    price: float
    stock: int = 0
```

**Now try the same bad data:**

```python
product = Product(name=99999, price="not a number", stock=-50)
```

Output:
```
pydantic_core._pydantic_core.ValidationError: 2 validation errors for Product
price
  Input should be a valid number, unable to parse string
  as a number [type=float_parsing, input_value='not a number', input_type=str]
name
  Input should be a valid string [type=string_type, input_value=99999, input_type=int]
```

**Two things just happened:**

1. **It caught BOTH errors at once** — not one at a time
2. **It told you exactly what's wrong** — which field, what was expected, what was received

**Now try data that's close but not quite right:**

```python
# String "25" for a float field — Pydantic is SMART about this
product = Product(name="Widget", price="25", stock="10")

print(product.price)  # 25.0  ← Coerced string "25" to float
print(product.stock)  # 10    ← Coerced string "10" to int

print(type(product.price))  # <class 'float'>
print(type(product.stock))  # <class 'int'>
```

> "Pydantic doesn't just reject bad data. It intelligently COERCES data when the conversion is sensible. A string '25' can become a float. A string 'not a number' cannot."

---

## 1.4 The Gatekeeper Mental Model

**Pydantic is the GATEKEEPER of your API.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE GATEKEEPER ANALOGY                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your API is a building. Anyone can walk up to the door         │
│  and try to come in. The internet sends ANYTHING.               │
│                                                                 │
│  WITHOUT PYDANTIC:                                              │
│  ─────────────────                                              │
│  The door is wide open. Anyone walks in with whatever           │
│  they're carrying. You find out something's wrong LATER —       │
│  when your code crashes, your database corrupts, or your        │
│  response makes no sense.                                       │
│                                                                 │
│  WITH PYDANTIC:                                                 │
│  ──────────────                                                 │
│  There's a gatekeeper at the door who:                          │
│                                                                 │
│  1. Checks identity (TYPE COERCION)                             │
│     "Show me your ID" → "25" → int 25 ✅                        │
│     "Show me your ID" → "abc" → NOT a number ❌                 │
│                                                                 │
│  2. Enforces rules (FIELD CONSTRAINTS)                          │
│     "Price must be positive" → -5 ❌                             │
│     "Name must be 1-100 chars" → "" ❌                           │
│                                                                 │
│  3. Does special inspections (CUSTOM VALIDATORS)                │
│     "If express shipping, must have phone number" ❌             │
│     "Email must be lowercase" → auto-corrected ✅               │
│                                                                 │
│  4. Gives different badges (REQUEST vs RESPONSE MODELS)         │
│     Incoming visitors show everything (password, etc.)          │
│     Outgoing visitors get a public badge (no password!)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The validation pipeline — how data flows through the gatekeeper:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE VALIDATION PIPELINE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Raw Data (JSON from client)                                   │
│       │                                                         │
│       ▼                                                         │
│   ┌───────────────────────┐                                     │
│   │    Type Coercion      │  "25" → 25, "true" → True          │
│   │    (Smart Parsing)    │                                     │
│   └───────────┬───────────┘                                     │
│               │                                                 │
│               ▼                                                 │
│   ┌───────────────────────┐                                     │
│   │   Type Validation     │  Is it actually int? str?           │
│   │   (Type Check)        │  Wrong type → Error                 │
│   └───────────┬───────────┘                                     │
│               │                                                 │
│               ▼                                                 │
│   ┌───────────────────────┐                                     │
│   │  Field Constraints    │  gt=0? min_length=1?                │
│   │  (Rule Check)         │  Violated → Error                   │
│   └───────────┬───────────┘                                     │
│               │                                                 │
│               ▼                                                 │
│   ┌───────────────────────┐                                     │
│   │  Custom Validators    │  Business logic checks              │
│   │  (Special Inspection) │                                     │
│   └───────────┬───────────┘                                     │
│               │                                                 │
│               ▼                                                 │
│   Validated Model Instance ✅                                    │
│   (Guaranteed correct. Safe to use everywhere.)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to code:**

```
Gatekeeper Concept       │  Pydantic
─────────────────────────│─────────────────────────
The building             │  Your API endpoint
Visitors arriving        │  Incoming JSON data
The gate / door          │  BaseModel class
ID check                 │  Type coercion & validation
Building rules           │  Field constraints (Field())
Special inspections      │  @field_validator, @model_validator
Different badges         │  Request vs Response models
Building policy          │  model_config (ConfigDict)
What you tell visitors   │  Serialization (.model_dump())
  vs what you file       │
```

---

# PART 2: BASEMODEL FUNDAMENTALS

## 2.1 Your First Model (Schema as Class)

**Connection to what you've learned:**

> "Remember dataclasses from Week 1, Lecture 2? BaseModel looks almost identical. Same syntax, radically different behavior."

```python
from pydantic import BaseModel

class Product(BaseModel):
    name: str
    price: float
    category: str
    description: str = ""     # Optional with default
    stock: int = 0            # Optional with default
```

**Compare side by side:**

```
┌─────────────────────────────────────────────────────────────────┐
│               DATACLASS vs BASEMODEL                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  from dataclasses import dataclass  │  from pydantic import     │
│                                     │      BaseModel            │
│  @dataclass                         │                           │
│  class Product:                     │  class Product(BaseModel):│
│      name: str                      │      name: str            │
│      price: float                   │      price: float         │
│      category: str                  │      category: str        │
│      description: str = ""          │      description: str = ""│
│      stock: int = 0                 │      stock: int = 0       │
│                                     │                           │
│  Stores anything.                   │  Validates everything.    │
│  Types are hints.                   │  Types are rules.         │
│                                     │                           │
└─────────────────────────────────────────────────────────────────┘
```

**Using the model — same as a dataclass:**

```python
# Create a product
product = Product(
    name="Mechanical Keyboard",
    price=89.99,
    category="electronics"
)

# Access fields — exactly like a dataclass
print(product.name)         # Mechanical Keyboard
print(product.price)        # 89.99
print(product.description)  # "" (default value)
print(product.stock)        # 0 (default value)
```

**Connection to type hints (Lecture 1, Week 1):**

> "See the type annotations — `name: str`, `price: float`? These are the SAME type hints you learned in Week 1. In a dataclass, they're documentation. In Pydantic, they're enforced rules. Your type hint knowledge is now doing REAL work."

---

## 2.2 Type Coercion (Smart Parsing)

**Pydantic doesn't just validate — it COERCES when reasonable.**

```python
# JSON from a web form often sends numbers as strings
product = Product(
    name="Widget",
    price="49.99",     # String, not float!
    category="food",
    stock="100"        # String, not int!
)

print(product.price)        # 49.99
print(type(product.price))  # <class 'float'>
print(product.stock)        # 100
print(type(product.stock))  # <class 'int'>
```

**Pydantic's coercion rules:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    TYPE COERCION RULES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Target Type     │  Input           │  Result                   │
│  ────────────────│──────────────────│───────────────────        │
│  int             │  "42"            │  42 ✅                    │
│  int             │  42.0            │  42 ✅                    │
│  int             │  "42.5"          │  ❌ Error                 │
│  int             │  "hello"         │  ❌ Error                 │
│                  │                  │                           │
│  float           │  "3.14"          │  3.14 ✅                  │
│  float           │  42              │  42.0 ✅                  │
│  float           │  "inf"           │  ❌ Error (by default)    │
│                  │                  │                           │
│  str             │  "hello"         │  "hello" ✅               │
│  str             │  42              │  ❌ Error (not coerced!)  │
│                  │                  │                           │
│  bool            │  true            │  True ✅                  │
│  bool            │  1               │  True ✅                  │
│  bool            │  "yes"           │  ❌ Error                 │
│                  │                  │                           │
└─────────────────────────────────────────────────────────────────┘
```

**Key insight — str is strict in Pydantic v2:**

```python
class Example(BaseModel):
    name: str

# ❌ This FAILS in Pydantic v2 — int is NOT auto-coerced to str
example = Example(name=42)
# ValidationError: Input should be a valid string
#   [type=string_type, input_value=42, input_type=int]
```

> "Pydantic v2 made a deliberate choice: converting 42 to '42' is often a bug, not an intention. An int in a name field? That's probably wrong. Pydantic protects you."

**Gatekeeper analogy:**

> "The gatekeeper checks your ID. If your driver's license says '25' in text, they understand you're 25 years old — that's reasonable coercion. But if you hand them a library card when they asked for a driver's license, that's a rejection. Different document, not just a different format."

---

## 2.3 Validation Errors (Rejecting Bad Data)

**When the gatekeeper says NO — what does it look like?**

```python
from pydantic import ValidationError

try:
    product = Product(
        name="",          # Empty string
        price=-10,        # Negative price (valid float, but wrong)
        category=42       # Wrong type
    )
except ValidationError as e:
    print(e)
```

Output:
```
2 validation errors for Product
category
  Input should be a valid string
    [type=string_type, input_value=42, input_type=int]
price
  Input should be a valid number
    [type=float_type, ...]
```

**Pydantic collects ALL errors, not just the first:**

```
┌─────────────────────────────────────────────────────────────────┐
│               MANUAL VALIDATION VS PYDANTIC                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MANUAL:                                                        │
│  ───────                                                        │
│  Client sends bad data                                          │
│  Server: "name is required"         ← Fix it, resubmit         │
│  Server: "price must be positive"   ← Fix it, resubmit         │
│  Server: "category is invalid"      ← Fix it, resubmit         │
│  3 round trips to discover 3 problems!                          │
│                                                                 │
│  PYDANTIC:                                                      │
│  ────────                                                       │
│  Client sends bad data                                          │
│  Server: "Here are ALL 3 problems at once."                     │
│  1 round trip. Client fixes everything. Done.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Structured errors — machine-readable for frontends:**

```python
try:
    product = Product(name="", price="abc", category=42)
except ValidationError as e:
    # Get errors as a list of dicts
    print(e.errors())
```

```python
[
    {
        "type": "string_type",
        "loc": ("category",),       # WHERE the error is
        "msg": "Input should be a valid string",  # WHAT went wrong
        "input": 42                  # WHAT was received
    },
    {
        "type": "float_parsing",
        "loc": ("price",),
        "msg": "Input should be a valid number, ...",
        "input": "abc"
    }
]
```

**Why this matters for APIs:**

> "Remember HTTP error responses from Lecture 1 this week? A good API returns structured errors — which field failed, why, and what was sent. Pydantic gives you this for FREE. You'll use `e.errors()` in your custom exception handlers."

---

## 2.4 Field Constraints (The Precision Rules)

**The gatekeeper doesn't just check types — it checks RULES.**

**`Field()` adds constraints beyond just the type:**

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    price: float = Field(gt=0, description="Price in USD")
    category: str
    description: str = Field(default="", max_length=1000)
    stock: int = Field(default=0, ge=0)
```

**Compare: our 40 lines of manual validation is now 5 lines of declarations.**

**Numeric constraints:**

```python
from pydantic import BaseModel, Field

class OrderItem(BaseModel):
    quantity: int = Field(gt=0, le=99)         # 1–99
    unit_price: float = Field(gt=0)            # Must be positive
    discount_percent: float = Field(ge=0, lt=100)  # 0–99.99
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 NUMERIC CONSTRAINTS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Constraint   │  Meaning                │  Example              │
│  ─────────────│─────────────────────────│──────────────         │
│  gt=0         │  Greater than 0         │  0.01 ✅  0 ❌        │
│  ge=0         │  Greater or equal to 0  │  0 ✅     -1 ❌       │
│  lt=100       │  Less than 100          │  99 ✅    100 ❌      │
│  le=100       │  Less or equal to 100   │  100 ✅   101 ❌      │
│               │                         │                       │
│  Combine them:│                         │                       │
│  gt=0, le=99  │  Between 1 and 99       │  50 ✅    0 ❌        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**String constraints:**

```python
class UserRegistration(BaseModel):
    username: str = Field(min_length=3, max_length=20)
    email: str = Field(min_length=5)
    zipcode: str = Field(pattern=r"^\d{5}$")  # Exactly 5 digits
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  STRING CONSTRAINTS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Constraint       │  Meaning                    │  Example       │
│  ─────────────────│─────────────────────────────│──────────      │
│  min_length=3     │  At least 3 characters      │  "ab" ❌       │
│  max_length=20    │  At most 20 characters       │  "a"*21 ❌     │
│  pattern=r"..."   │  Must match regex pattern    │  "12345" ✅    │
│                   │                              │  "abcde" ❌    │
│                                                                 │
│  NOTE: The roadmap says "regex" — that was the Pydantic v1      │
│  name. In Pydantic v2, the parameter is called "pattern".       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Field() also provides documentation:**

```python
class Product(BaseModel):
    name: str = Field(
        min_length=1,
        max_length=100,
        description="Product display name",
        examples=["Mechanical Keyboard", "USB-C Cable"]
    )
    price: float = Field(
        gt=0,
        description="Price in USD, must be positive"
    )
```

> "Remember Swagger UI from Lecture 2 this week? The `description` and `examples` you put in `Field()` show up AUTOMATICALLY in your API docs. Your code IS your documentation."

---

## 2.5 Optional Fields and Defaults

**Three patterns you'll use constantly:**

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    # REQUIRED — no default, must be provided
    name: str
    
    # OPTIONAL WITH DEFAULT VALUE — uses the default if not provided
    stock: int = 0
    description: str = ""
    
    # OPTIONAL / NULLABLE — can be explicitly set to None
    discount_price: float | None = None
    
    # REQUIRED but can be None — must be provided, but None is valid
    external_id: str | None
```

**Visualize the differences:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  FIELD OPTIONALITY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PATTERN              │  Must provide?  │  Can be None?          │
│  ─────────────────────│─────────────────│────────────────        │
│  name: str            │  YES            │  NO                    │
│  stock: int = 0       │  NO (default 0) │  NO                    │
│  price: float | None  │  NO (def None)  │  YES                   │
│    = None             │                 │                        │
│  ext_id: str | None   │  YES            │  YES                   │
│                                                                 │
│  Recall from Week 1 Lecture 1:                                  │
│  float | None is the same as Optional[float]                    │
│  Both mean "float or None"                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Mutable defaults — use Field(default_factory=...):**

```python
# ❌ WRONG — mutable default shared across all instances
class BadModel(BaseModel):
    tags: list[str] = []  # Pydantic actually handles this safely,
                          # but it's good practice to be explicit

# ✅ BETTER — explicitly create a new list each time
class GoodModel(BaseModel):
    tags: list[str] = Field(default_factory=list)
```

> "Pydantic actually protects you from the mutable default trap that bites regular Python classes. But using `default_factory` is explicit and makes your intent clear — the same good habit you learned with dataclasses in Week 1."
---
## 2.6 Built-in Special Types

**Connection to what you've learned:**

> "In section 2.4, you used `pattern=r"..."` in `Field()` to validate strings against a regex. Writing a correct email regex is notoriously hard — there are over 20 edge cases. Writing a correct URL regex is even harder. Pydantic ships validated types for these common needs. You never have to write those regexes."

**First: install the email extra:**

```
# Email validation requires an optional dependency
uv add pydantic[email]
```

```python
from pydantic import BaseModel, EmailStr, HttpUrl, AnyUrl

class UserProfile(BaseModel):
    email: EmailStr               # Validates email format automatically
    website: HttpUrl | None = None  # Validates http/https URL
    portfolio: AnyUrl | None = None # Any URL (any scheme accepted)
```

**Using it:**

```python
# ✅ Valid
user = UserProfile(
    email="alice@example.com",
    website="https://alice.dev/portfolio",
)

print(user.email)            # alice@example.com     ← plain string
print(user.website)          # https://alice.dev/portfolio  ← Url object!
print(type(user.website))    # <class 'pydantic_core.Url'>
print(str(user.website))     # https://alice.dev/portfolio  ← convert to string

# ❌ Invalid email
user = UserProfile(email="not-an-email")
# ValidationError: value is not a valid email address

# ❌ Invalid URL — missing scheme
user = UserProfile(email="a@b.com", website="alice.dev")
# ValidationError: URL scheme should be 'http' or 'https'
```

**IMPORTANT — HttpUrl is a Url object, not a string:**

```python
user = UserProfile(email="a@b.com", website="https://alice.dev/path?q=1#anchor")

# Decompose the URL into parts
print(user.website.scheme)    # https
print(user.website.host)      # alice.dev
print(user.website.path)      # /path
print(user.website.query)     # q=1
print(user.website.fragment)  # anchor

# When you serialise it — Pydantic converts it to a string automatically
print(user.model_dump())
# {"email": "alice@example.com", "website": "https://alice.dev/path?q=1#anchor"}
```

> "The Url object is useful when your business logic needs to inspect parts of a URL — for example, blocking requests from certain hosts. For serialization, Pydantic handles the string conversion automatically. You only need to call `str()` manually when passing it to a function that expects a plain string."

**Other built-in special types:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  BUILT-IN SPECIAL TYPES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Type               │  What it validates           │  Package   │
│  ───────────────────│──────────────────────────────│──────────  │
│  EmailStr           │  email format                │  pydantic  │
│                     │                              │  [email]   │
│  HttpUrl            │  http/https URL (strict)     │  pydantic  │
│  AnyUrl             │  any URL with a scheme       │  pydantic  │
│  AnyHttpUrl         │  http/https (less strict)    │  pydantic  │
│  IPvAnyAddress      │  IPv4 or IPv6 address        │  pydantic  │
│  PastDate           │  date that is in the past    │  pydantic  │
│  FutureDate         │  date that is in the future  │  pydantic  │
│  PositiveInt        │  int > 0  (same as gt=0)     │  pydantic  │
│  NonNegativeInt     │  int >= 0 (same as ge=0)     │  pydantic  │
│  PositiveFloat      │  float > 0                   │  pydantic  │
│  NegativeFloat      │  float < 0                   │  pydantic  │
│                                                                 │
│  All except EmailStr are included with pydantic by default.    │
│  EmailStr requires:  uv add pydantic[email]                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**These are shorthand for Annotated types you already know:**

```python
from pydantic import BaseModel, PositiveInt, PositiveFloat, NonNegativeInt

# These two models are EQUIVALENT:

class InventoryVerbose(BaseModel):
    quantity: int = Field(ge=0)       # NonNegativeInt
    price: float = Field(gt=0)        # PositiveFloat
    restock_at: int = Field(gt=0)     # PositiveInt

class InventoryConcise(BaseModel):
    quantity: NonNegativeInt
    price: PositiveFloat
    restock_at: PositiveInt
```

> "Use the named types when the name describes intent clearly. Use `Field(gt=0)` when you need different bounds (e.g., `Field(ge=5, le=100)`). They produce identical validation — the named types are just readable aliases."

**Swagger UI benefit:**

> "When you use `EmailStr`, Swagger UI shows the field with `format: email`. When you use `HttpUrl`, it shows `format: uri`. Your API documentation becomes accurate without any extra effort."

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 2.6                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Build a ContactCard model with the following fields:           │
│  - email: required, must be a valid email                       │
│  - linkedin_url: optional, must be a valid http/https URL       │
│  - age: required, must be a positive integer                    │
│                                                                 │
│  After creating a valid instance, inspect:                      │
│  1. What is the Python type of linkedin_url?                    │
│  2. How do you access only the host part of the URL?            │
│  3. Does model_dump() give you a string or the Url object?      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

from pydantic import BaseModel, EmailStr, HttpUrl, PositiveInt

class ContactCard(BaseModel):
    email: EmailStr
    linkedin_url: HttpUrl | None = None
    age: PositiveInt

card = ContactCard(
    email="alice@example.com",
    linkedin_url="https://linkedin.com/in/alice",
    age=30
)

# 1. linkedin_url is a Url OBJECT, not a plain string
print(type(card.linkedin_url))         # <class 'pydantic_core.Url'>

# 2. Access the host component
print(card.linkedin_url.host)          # linkedin.com

# 3. model_dump() serialises Url objects to strings automatically
data = card.model_dump()
print(data["linkedin_url"])            # https://linkedin.com/in/alice
print(type(data["linkedin_url"]))      # <class 'str'>  ← a plain string!

# ❌ age=0 is not positive
# ContactCard(email="a@b.com", age=0)  → ValidationError

# ❌ Missing scheme
# ContactCard(email="a@b.com", linkedin_url="linkedin.com/in/alice", age=1)
# → ValidationError: URL scheme should be 'http' or 'https'
```

---

## 2.7 Enums with Pydantic

**Connection to what you've learned:**

> "Go back to section 1.1, the Manual Validation Nightmare. Remember this line: `if data["category"] not in ["electronics", "clothing", "food"]`? That's the string-based approach. It has two silent failure modes: you typo `'electroncs'` and it becomes an undetected bug, and nothing in your editor or type checker can warn you. Python Enums solve both."

**Define a string enum:**

```python
from enum import Enum

class Category(str, Enum):
    electronics = "electronics"
    clothing = "clothing"
    food = "food"
```

**Why `str, Enum` instead of just `Enum`?**

```
┌─────────────────────────────────────────────────────────────────┐
│                class Enum  vs  class (str, Enum)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  class Category(Enum):         │  class Category(str, Enum):   │
│      electronics = "electronics"│    electronics = "electronics"│
│                                 │                               │
│  # Serialisation breaks:        │  # Serialisation just works:  │
│  json.dumps(Category.electronics│  json.dumps(Category.electronics│
│  → TypeError!                   │  → '"electronics"'            │
│                                 │                               │
│  f"{Category.electronics}"      │  f"{Category.electronics}"   │
│  → "Category.electronics" ❌    │  → "electronics" ✅           │
│                                 │                               │
│  Category.electronics ==        │  Category.electronics ==      │
│      "electronics"              │      "electronics"            │
│  → False ❌                     │  → True ✅                    │
│                                 │                               │
│                Always use (str, Enum) for API-facing fields.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Using Enum in a Pydantic model:**

```python
from pydantic import BaseModel, Field
from enum import Enum

class Category(str, Enum):
    electronics = "electronics"
    clothing = "clothing"
    food = "food"

class Priority(str, Enum):
    low = "low"
    medium = "medium"
    high = "high"
    critical = "critical"

class Product(BaseModel):
    name: str = Field(min_length=1)
    category: Category
    priority: Priority = Priority.medium    # Default is the enum member
```

**Pydantic accepts both the raw string AND the enum member:**

```python
# Both of these work — Pydantic coerces the string to the enum
p1 = Product(name="Widget", category="electronics", priority="high")
p2 = Product(name="Widget", category=Category.electronics, priority=Priority.high)

print(p1.category)                      # Category.electronics
print(p1.category.value)                # electronics
print(type(p1.category))                # <class 'Category'>
print(p1.category == "electronics")     # True  ← str Enum!
```

> "Your API receives raw JSON strings. Pydantic coerces them to enum members. Inside your code, you work with enum members — with IDE autocomplete and type safety. The two layers never collide."

**Invalid input gives a self-documenting error:**

```python
product = Product(name="Widget", category="furniture", priority="high")
# ValidationError: 1 validation error for Product
# category
#   Input should be 'electronics', 'clothing' or 'food'
#   [type=enum, input_value='furniture', input_type=str]
```

> "The error message lists all valid values automatically, for free. No extra documentation effort on your part."

**Serialization:**

```python
product = Product(name="Widget", category=Category.electronics, priority=Priority.high)

print(product.model_dump())
# {"name": "Widget", "category": "electronics", "priority": "high"}
# ← Enum members serialise as their VALUES, not as enum representations

print(product.model_dump_json())
# {"name":"Widget","category":"electronics","priority":"high"}
```

**Integer enums — same pattern:**

```python
class HttpStatusGroup(int, Enum):
    success = 200
    not_found = 404
    server_error = 500

class ApiResult(BaseModel):
    status: HttpStatusGroup
    body: str

result = ApiResult(status=200, body="OK")
print(result.status)         # HttpStatusGroup.success
print(result.status.value)   # 200
print(result.model_dump())   # {"status": 200, "body": "OK"}
```

> "Use `(int, Enum)` for numeric enums — they serialise as their integer values. Use `(str, Enum)` for string enums. The pattern is always the same: the base type first, then `Enum`."

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 2.7                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Define a TaskStatus enum (str, Enum) with four values:      │
│     todo, in_progress, done, cancelled.                         │
│                                                                 │
│  2. Build a Task model: title (str), status (TaskStatus),       │
│     with status defaulting to todo.                             │
│                                                                 │
│  3. Create a task with status="in_progress" (a raw string).     │
│     Does it work? What is the Python type of task.status?       │
│                                                                 │
│  4. What error message appears when you pass status="finished"? │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

from enum import Enum
from pydantic import BaseModel

class TaskStatus(str, Enum):
    todo = "todo"
    in_progress = "in_progress"
    done = "done"
    cancelled = "cancelled"

class Task(BaseModel):
    title: str
    status: TaskStatus = TaskStatus.todo

# 3. String input is coerced to the enum member
task = Task(title="Write tests", status="in_progress")
print(task.status)                    # TaskStatus.in_progress
print(type(task.status))              # <class 'TaskStatus'>
print(task.status == "in_progress")   # True  ← because str, Enum

# Default works
task2 = Task(title="Review PR")
print(task2.status)                   # TaskStatus.todo

# 4. Invalid value — lists all valid options automatically
try:
    Task(title="Test", status="finished")
except Exception as e:
    print(e)
# ValidationError:
# status
#   Input should be 'todo', 'in_progress', 'done' or 'cancelled'
```

---

## 2.8 TypeAdapter — Validating Without a Model

**The problem: you want to validate a single value, but creating a whole BaseModel for one field is wasteful.**

```python
# You want to validate a list of integers from a query parameter.
# This works, but it's clunky:
class _IntListWrapper(BaseModel):
    values: list[int]

def parse_ids(raw_list: list) -> list[int]:
    return _IntListWrapper(values=raw_list).values  # Wrap just to unwrap
```

**TypeAdapter is the direct solution:**

```python
from pydantic import TypeAdapter

# Create an adapter for any type expression
int_list_adapter = TypeAdapter(list[int])

# Validate a Python value (lax mode — coerces "1" → 1)
result = int_list_adapter.validate_python(["1", "2", "3"])
print(result)   # [1, 2, 3]

# Validate a JSON string
result = int_list_adapter.validate_json("[1, 2, 3]")
print(result)   # [1, 2, 3]

# Invalid — raises ValidationError just like a model would
int_list_adapter.validate_python(["1", "abc", "3"])
# ValidationError: 1 validation error for list[int]
# 1
#   Input should be a valid integer, unable to parse string as integer
```

**Where you use this in real API code:**

```python
from pydantic import TypeAdapter
from typing import Literal

# Validate a query parameter that must be one of two values
sort_order_adapter = TypeAdapter(Literal["asc", "desc"])

sort_order_adapter.validate_python("asc")     # ✅ "asc"
sort_order_adapter.validate_python("random")  # ❌ ValidationError

# Validate a list of enum values from a query parameter
from enum import Enum

class Tag(str, Enum):
    python = "python"
    backend = "backend"
    api = "api"

tag_list_adapter = TypeAdapter(list[Tag])
tags = tag_list_adapter.validate_python(["python", "backend"])
print(tags)   # [<Tag.python: 'python'>, <Tag.backend: 'backend'>]
```

**TypeAdapter also serialises:**

```python
from datetime import datetime
from pydantic import TypeAdapter

dt_adapter = TypeAdapter(datetime)

# Validate
dt = dt_adapter.validate_python("2025-01-15T10:30:00")
print(dt)   # datetime(2025, 1, 15, 10, 30)

# Serialise (to Python)
print(dt_adapter.dump_python(dt))     # datetime(2025, 1, 15, 10, 30)

# Serialise (to JSON bytes)
print(dt_adapter.dump_json(dt))       # b'"2025-01-15T10:30:00"'
```

**Performance tip — define at module level:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TYPEADAPTER: MODULE-LEVEL DEFINITION               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  # ❌ SLOW — schema is rebuilt on every function call           │
│  def validate_tags(raw: list) -> list[Tag]:                     │
│      adapter = TypeAdapter(list[Tag])   ← rebuilt every call!  │
│      return adapter.validate_python(raw)                        │
│                                                                 │
│  # ✅ FAST — schema built once, adapter reused                  │
│  _tag_adapter = TypeAdapter(list[Tag])  ← defined once!        │
│                                                                 │
│  def validate_tags(raw: list) -> list[Tag]:                     │
│      return _tag_adapter.validate_python(raw)                   │
│                                                                 │
│  TypeAdapter builds a full validation schema internally.        │
│  Creating it inside a loop or function is an expensive mistake. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**TypeAdapter vs BaseModel — when to use which:**

```
┌─────────────────────────────────────────────────────────────────┐
│             TypeAdapter  vs  BaseModel                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Use BaseModel when:              │  Use TypeAdapter when:      │
│  ─────────────────────────────────│──────────────────────────   │
│  You have multiple fields         │  One value or one type      │
│  You need cross-field validators  │  Validating a list or dict  │
│  FastAPI request / response body  │  Query / path / header      │
│  The schema is part of your API   │  Parsing config values      │
│  contract                         │  Testing a type expression  │
│                                   │  Ad-hoc validation          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 2.8                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A search endpoint receives a comma-separated string of IDs     │
│  as a query parameter, e.g., "1,2,3,4".                         │
│                                                                 │
│  Using TypeAdapter, write a function parse_id_param(raw: str)   │
│  that splits the string, validates each part as a PositiveInt,  │
│  and returns a list[int].                                        │
│                                                                 │
│  What error does it raise for the input "1,0,3"?                │
│  What about "1,abc,3"?                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

from pydantic import TypeAdapter, PositiveInt

# Define adapter once at module level
_id_list_adapter = TypeAdapter(list[PositiveInt])

def parse_id_param(raw: str) -> list[int]:
    parts = [part.strip() for part in raw.split(",")]
    return _id_list_adapter.validate_python(parts)

# ✅ Valid
print(parse_id_param("1,2,3,4"))   # [1, 2, 3, 4]

# ❌ Input "1,0,3" — 0 is not a PositiveInt (must be > 0)
# ValidationError: 1 validation error for list[int]
# 1  (index 1 in the list)
#   Input should be greater than 0

# ❌ Input "1,abc,3" — "abc" cannot parse to int
# ValidationError: 1 validation error for list[int]
# 1
#   Input should be a valid integer, unable to parse string as integer
```

---

## 2.9 model_validate() and model_validate_json()

**You already know `Product(name="Widget", price=9.99)`. There are two class-method alternatives — and they matter when data arrives from the outside world.**

```python
from pydantic import BaseModel

class Product(BaseModel):
    name: str
    price: float
    stock: int = 0

# Method 1: Direct instantiation — you know the fields at write time
p1 = Product(name="Widget", price=9.99)

# Method 2: model_validate() — data is already a Python dict or object
p2 = Product.model_validate({"name": "Widget", "price": 9.99})

# Method 3: model_validate_json() — data is a raw JSON string or bytes
p3 = Product.model_validate_json('{"name": "Widget", "price": 9.99}')
```

**All three produce an identical validated model. The difference is WHERE the data starts.**

```
┌─────────────────────────────────────────────────────────────────┐
│           WHICH VALIDATION METHOD TO USE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Product(field=value, ...)                                      │
│  ────────────────────────                                       │
│  Source: your own code — you type the values.                   │
│  Use for: creating models in tests, seeds, factories.           │
│                                                                 │
│  Product.model_validate(data)                                   │
│  ─────────────────────────────                                  │
│  Source: data is already a Python dict or an ORM object.        │
│  Use for: ORM objects (from_attributes=True), dicts from        │
│  another library, data already parsed from elsewhere.           │
│                                                                 │
│  Product.model_validate_json(json_string)                       │
│  ─────────────────────────────────────────                      │
│  Source: a raw JSON string or bytes from the network/file.      │
│  Use for: file.read(), webhook payload bodies, queue messages.  │
│                                                                 │
│  PERFORMANCE:                                                   │
│  model_validate_json() is FASTER than json.loads() + validate() │
│  It uses Pydantic's Rust-based JSON parser — one pass, not two. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical example — webhook handler:**

```python
import json
from pydantic import BaseModel

class WebhookEvent(BaseModel):
    event_type: str
    resource_id: int
    timestamp: str

raw_body = '{"event_type": "order.paid", "resource_id": 42, "timestamp": "2025-01-01T00:00:00Z"}'

# ✅ One pass — Pydantic parses JSON and validates in a single step
event = WebhookEvent.model_validate_json(raw_body)

# ❌ Two passes — parse JSON first, then validate: slower, no benefit
event = WebhookEvent.model_validate(json.loads(raw_body))
```

**Both raise the same ValidationError for bad data:**

```python
# These produce identical errors
Product.model_validate({"name": 999, "price": "oops"})
Product.model_validate_json('{"name": 999, "price": "oops"}')
# Both: ValidationError: price → Input should be a valid number
```

**Common mistake — passing a JSON string to model_validate():**

```python
# ❌ This FAILS — model_validate expects a dict, not a string
Product.model_validate('{"name": "Widget", "price": 9.99}')
# ValidationError: Input should be a valid dictionary or instance of Product
#   [type=model_type, input_value='{"name": "Widget"...', input_type=str]

# ✅ Correct — for raw JSON strings, use model_validate_json()
Product.model_validate_json('{"name": "Widget", "price": 9.99}')
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 2.9                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  class AppConfig(BaseModel):                                    │
│      db_url: str                                                │
│      debug: bool = False                                        │
│      port: int = 8000                                           │
│                                                                 │
│  For each function below, which validation method should it use?│
│  Justify your answer.                                           │
│                                                                 │
│  1. def from_file(path: str) -> AppConfig:                      │
│       with open(path) as f:                                     │
│           data = f.read()   # raw string from disk              │
│       return AppConfig.???(data)                                │
│                                                                 │
│  2. def from_dict(data: dict) -> AppConfig:                     │
│       return AppConfig.???(data)                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

from pydantic import BaseModel

class AppConfig(BaseModel):
    db_url: str
    debug: bool = False
    port: int = 8000

# 1. f.read() returns a raw string → use model_validate_json()
#    It goes directly from JSON string to a validated model in one step.
def from_file(path: str) -> AppConfig:
    with open(path) as f:
        return AppConfig.model_validate_json(f.read())

# 2. data is already a Python dict → use model_validate()
def from_dict(data: dict) -> AppConfig:
    return AppConfig.model_validate(data)

# ❌ Common mistake: from_dict using validate_json would fail
# AppConfig.model_validate_json({"db_url": "..."})
# → TypeError: argument 'data' must be str or bytes-like, not 'dict'
```
---

# PART 3: NESTED MODELS

## 3.1 Models Inside Models (Composition)

**Real data isn't flat. An order has an address. A blog post has an author.**

```python
from pydantic import BaseModel, Field

class Address(BaseModel):
    street: str = Field(min_length=1)
    city: str = Field(min_length=1)
    country: str = Field(min_length=2, max_length=2)  # ISO code
    zipcode: str = Field(pattern=r"^\d{5}$")

class Customer(BaseModel):
    name: str = Field(min_length=1)
    email: str
    shipping_address: Address  # ← A model INSIDE a model
```

**Using it:**

```python
customer = Customer(
    name="Alice",
    email="alice@example.com",
    shipping_address={           # Pass a dict — Pydantic converts it!
        "street": "123 Main St",
        "city": "Springfield",
        "country": "US",
        "zipcode": "62704"
    }
)

print(customer.shipping_address.city)  # Springfield
print(type(customer.shipping_address)) # <class 'Address'>
```

**The key behavior — you pass a dict, Pydantic creates the nested model:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  NESTED MODEL PARSING                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What you SEND (JSON from client):                              │
│                                                                 │
│  {                                                              │
│    "name": "Alice",                                             │
│    "email": "alice@example.com",                                │
│    "shipping_address": {          ← Just a plain dict           │
│      "street": "123 Main St",                                   │
│      "city": "Springfield",                                     │
│      "country": "US",                                           │
│      "zipcode": "62704"                                         │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  What Pydantic CREATES:                                         │
│                                                                 │
│  Customer(                                                      │
│    name="Alice",                                                │
│    email="alice@example.com",                                   │
│    shipping_address=Address(      ← Validated Address object!   │
│      street="123 Main St",                                      │
│      city="Springfield",                                        │
│      country="US",                                              │
│      zipcode="62704"                                            │
│    )                                                            │
│  )                                                              │
│                                                                 │
│  Every level validated. Every rule checked. Automatically.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Gatekeeper analogy:**

> "A nested model is like a checkpoint inside a checkpoint. The outer gatekeeper checks your name and email. Then sends your shipping address through ANOTHER gatekeeper who checks street, city, country, zipcode. Each level has its own rules."

---

## 3.2 Collections of Models

**Connection to type hints (Lecture 1, Week 1):**

> "Remember typing collection types — `list[int]`, `dict[str, float]`? They work with Pydantic models too."

```python
class OrderItem(BaseModel):
    product_name: str
    quantity: int = Field(gt=0)
    unit_price: float = Field(gt=0)

class Order(BaseModel):
    customer: Customer                    # Single nested model
    items: list[OrderItem]                # List of models
    notes: dict[str, str] = {}            # Dict with type params
```

**Using it:**

```python
order = Order(
    customer={
        "name": "Alice",
        "email": "alice@example.com",
        "shipping_address": {
            "street": "123 Main St",
            "city": "Springfield",
            "country": "US",
            "zipcode": "62704"
        }
    },
    items=[
        {"product_name": "Keyboard", "quantity": 1, "unit_price": 89.99},
        {"product_name": "Mouse", "quantity": 2, "unit_price": 29.99}
    ],
    notes={"gift": "Wrap in blue paper"}
)

# Every item is validated individually
print(order.items[0].product_name)  # Keyboard
print(type(order.items[0]))         # <class 'OrderItem'>
print(len(order.items))             # 2
```

**What happens with one bad item?**

```python
order = Order(
    customer={...},
    items=[
        {"product_name": "Keyboard", "quantity": 1, "unit_price": 89.99},
        {"product_name": "Mouse", "quantity": -3, "unit_price": 29.99},
        #                          ^^^ Invalid! quantity must be > 0
    ]
)
# ValidationError: 1 validation error for Order
# items.1.quantity
#   Input should be greater than 0 [...]
```

**Notice the error location: `items.1.quantity`** — Pydantic tells you the EXACT path to the problem: the `quantity` field of item at index `1` in the `items` list.

---

## 3.3 Deep Validation (Errors Point to the Exact Spot)

**Pydantic errors include a `loc` (location) that traces the path:**

```python
try:
    order = Order(
        customer={
            "name": "",                  # Empty!
            "email": "alice@example.com",
            "shipping_address": {
                "street": "123 Main St",
                "city": "Springfield",
                "country": "USA",        # Too long! Max 2 chars
                "zipcode": "abc"         # Doesn't match pattern
            }
        },
        items=[
            {"product_name": "Keyboard", "quantity": 0, "unit_price": -5}
        ]
    )
except ValidationError as e:
    for error in e.errors():
        print(f"  Location: {error['loc']}")
        print(f"  Message:  {error['msg']}")
        print()
```

Output:
```
  Location: ('customer', 'name')
  Message:  String should have at least 1 character

  Location: ('customer', 'shipping_address', 'country')
  Message:  String should have at most 2 characters

  Location: ('customer', 'shipping_address', 'zipcode')
  Message:  String should match pattern '^\d{5}$'

  Location: ('items', 0, 'quantity')
  Message:  Input should be greater than 0

  Location: ('items', 0, 'unit_price')
  Message:  Input should be greater than 0
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    ERROR LOCATION PATH                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ('customer', 'shipping_address', 'zipcode')                    │
│       │              │                │                         │
│       │              │                └─ The field that failed   │
│       │              └─ Inside this nested model                │
│       └─ Inside this field                                      │
│                                                                 │
│  ('items', 0, 'unit_price')                                     │
│      │    │      │                                              │
│      │    │      └─ The field that failed                       │
│      │    └─ At list index 0                                    │
│      └─ Inside this field                                       │
│                                                                 │
│  This is like a GPS for your data errors.                       │
│  You know EXACTLY where to look.                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

## 3.4 Recursive / Self-Referencing Models

**Connection to what you've learned:**

> "In section 3.1, you nested `Address` inside `Customer`. Both models were fully defined before we used them. But what about a model that contains itself — like a comment that has replies, each of which is also a comment?"

**The pattern you need: trees and hierarchies.**

```
Comment
  └─ replies: list[Comment]   ← Each reply is also a Comment
       └─ replies: list[Comment]
            └─ ...  (arbitrary depth)
```

**Attempt 1 — fails, because Comment isn't fully defined yet:**

```python
from pydantic import BaseModel

class Comment(BaseModel):
    text: str
    author: str
    replies: list[Comment] = []    # ← NameError or schema error
                                   # Python hasn't finished defining Comment!
```

**The fix — forward reference using a string, then model_rebuild():**

```python
from pydantic import BaseModel

class Comment(BaseModel):
    text: str
    author: str
    replies: list["Comment"] = []   # ← String: "Comment" not yet resolved

# REQUIRED: tell Pydantic to resolve the forward reference NOW,
# after the class is fully defined.
Comment.model_rebuild()
```

**What happens if you forget model_rebuild():**

```python
class Comment(BaseModel):
    text: str
    replies: list["Comment"] = []
# model_rebuild() NOT called

comment = Comment(
    text="Hello",
    replies=[{"text": "Reply", "replies": []}]
)

# SILENT FAILURE — replies stay as plain dicts, not Comment objects!
print(type(comment.replies[0]))   # <class 'dict'>  ← WRONG
                                  # No validation, no type safety

# With model_rebuild():
Comment.model_rebuild()
comment = Comment(
    text="Hello",
    replies=[{"text": "Reply", "replies": []}]
)
print(type(comment.replies[0]))   # <class 'Comment'>  ← CORRECT
```

```
┌─────────────────────────────────────────────────────────────────┐
│               WHAT model_rebuild() DOES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  When Python parses the class body, "Comment" in the string     │
│  annotation is unresolved — the class isn't fully built yet.    │
│                                                                 │
│  model_rebuild() says: "The class is fully defined now.         │
│  Look up the real Comment class and rebuild the schema          │
│  using the actual type."                                        │
│                                                                 │
│  Without it:              │  With it:                          │
│  replies stay as dicts    │  replies are validated Comment      │
│  no nested validation     │  objects — full recursive check     │
│  no type safety           │  mypy understands the types         │
│                                                                 │
│  Also required for:                                             │
│  ✓ Two models that reference each other (mutual reference)      │
│  ✓ Model A imports Model B, Model B imports Model A             │
│  ✓ After dynamically adding fields to a model                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Full working example — comment thread:**

```python
from pydantic import BaseModel
from datetime import datetime

class Comment(BaseModel):
    text: str
    author: str
    created_at: datetime = None   # simplified
    replies: list["Comment"] = []

Comment.model_rebuild()

# Build a thread
thread = Comment(
    text="Great article on async patterns!",
    author="alice",
    replies=[
        Comment(
            text="Agreed! The gather() example was clear.",
            author="bob",
            replies=[
                Comment(text="Especially the timeout section.", author="carol")
            ]
        ),
        Comment(text="Could you cover trio next?", author="dave")
    ]
)

print(thread.replies[0].text)                 # Agreed! The gather()...
print(thread.replies[0].replies[0].text)      # Especially the timeout...
print(type(thread.replies[0]))                # <class 'Comment'>
print(type(thread.replies[0].replies[0]))     # <class 'Comment'>
```

**Mutual reference — two models that reference each other:**

```python
from pydantic import BaseModel

class Department(BaseModel):
    name: str
    employees: list["Employee"] = []

class Employee(BaseModel):
    name: str
    department: "Department | None" = None

# Rebuild BOTH after both are fully defined
Department.model_rebuild()
Employee.model_rebuild()

dept = Department(name="Engineering", employees=[
    Employee(name="Alice"),
    Employee(name="Bob"),
])
print(type(dept.employees[0]))   # <class 'Employee'>
```

> "Rule of thumb: every model that contains a forward reference — whether to itself or to another model defined later — needs `model_rebuild()` called after all referenced models are fully defined."

**Error you will see if you forget:**

```
pydantic.errors.PydanticUserError: `Comment` is not fully defined;
you should define `Comment`, then call `Comment.model_rebuild()`.
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 3.4                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Model an org chart:                                            │
│  - An Employee has a name (str) and a list of direct_reports    │
│    which are also Employees.                                     │
│                                                                 │
│  - Build a hierarchy: 1 CEO → 2 directors → 2 engineers each.  │
│                                                                 │
│  - Write a function count_headcount(e: Employee) -> int that    │
│    returns the total number of people in that subtree           │
│    (including the root employee).                               │
│                                                                 │
│  Deliberately skip model_rebuild(). What goes wrong?            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

from pydantic import BaseModel

class Employee(BaseModel):
    name: str
    direct_reports: list["Employee"] = []

# Without model_rebuild() first — let's see what breaks
emp = Employee(name="CEO", direct_reports=[{"name": "Director", "direct_reports": []}])
print(type(emp.direct_reports[0]))   # <class 'dict'>  ← silently wrong!

# Now fix it
Employee.model_rebuild()

def count_headcount(e: Employee) -> int:
    return 1 + sum(count_headcount(r) for r in e.direct_reports)

org = Employee(
    name="CEO",
    direct_reports=[
        Employee(name="Director A", direct_reports=[
            Employee(name="Engineer 1"),
            Employee(name="Engineer 2"),
        ]),
        Employee(name="Director B", direct_reports=[
            Employee(name="Engineer 3"),
            Employee(name="Engineer 4"),
        ]),
    ]
)

print(count_headcount(org))           # 7
print(type(org.direct_reports[0]))    # <class 'Employee'>  ← correct after rebuild
```

---

# PART 4: CUSTOM VALIDATORS

## 4.1 @field_validator — Single Field Logic

**Sometimes built-in constraints aren't enough. You need custom logic.**

**Connection to what you've learned:**

> "Remember decorators from Week 1, Lecture 2? `@field_validator` is a decorator. It works exactly like the decorators you already know — it wraps a function to add behavior. The `@` syntax should feel familiar now."

```python
from pydantic import BaseModel, Field, field_validator

class UserRegistration(BaseModel):
    username: str = Field(min_length=3, max_length=20)
    email: str
    age: int = Field(ge=0)
    
    @field_validator("username")
    @classmethod
    def username_must_be_alphanumeric(cls, v: str) -> str:
        if not v.isalnum():
            raise ValueError("username must contain only letters and numbers")
        return v

    @field_validator("email")
    @classmethod
    def email_must_contain_at(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("email must contain @")
        return v.lower()  # ← Normalize to lowercase!
```

**Using it:**

```python
# ✅ Valid — and email gets lowercased
user = UserRegistration(username="alice42", email="Alice@Example.COM", age=25)
print(user.email)  # alice@example.com ← Normalized!

# ❌ Invalid username
user = UserRegistration(username="alice!!", email="a@b.com", age=25)
# ValidationError: username must contain only letters and numbers

# ❌ Invalid email
user = UserRegistration(username="alice42", email="not-an-email", age=25)
# ValidationError: email must contain @
```

**The anatomy of a field validator:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 FIELD VALIDATOR ANATOMY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @field_validator("email")       ← Which field(s) to validate   │
│  @classmethod                    ← Required in Pydantic v2      │
│  def email_must_be_valid(        ← Name describes the rule      │
│      cls,                        ← Class (like a classmethod)   │
│      v: str                      ← The field's value            │
│  ) -> str:                       ← Must return the (maybe       │
│                                     transformed) value          │
│      if "@" not in v:                                           │
│          raise ValueError(...)   ← Raise ValueError to reject   │
│      return v.lower()            ← Return value (can modify!)   │
│                                                                 │
│  KEY RULES:                                                     │
│  1. Must have @classmethod under @field_validator               │
│  2. First param is cls (not self)                               │
│  3. Must RETURN a value (the validated/transformed field)       │
│  4. Raise ValueError to reject                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Validate multiple fields with the same validator:**

```python
class Product(BaseModel):
    name: str
    category: str
    description: str = ""
    
    @field_validator("name", "category", "description")
    @classmethod
    def strip_whitespace(cls, v: str) -> str:
        """Remove leading/trailing whitespace from all string fields"""
        return v.strip()
```

---

## 4.2 Validation Modes (before vs after)

**WHEN does your validator run? This matters.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  VALIDATION MODES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  mode="after" (DEFAULT)                                         │
│  ──────────────────────                                         │
│  Runs AFTER Pydantic's built-in type checking and coercion.     │
│  You receive the already-validated, correctly-typed value.       │
│                                                                 │
│     Raw input → [Type coercion] → [Constraints] → [YOUR CODE]  │
│                                                                 │
│  You get: the correct type (str, int, etc.) guaranteed.         │
│  Use when: You want to do extra checks on valid data.           │
│                                                                 │
│                                                                 │
│  mode="before"                                                  │
│  ─────────────                                                  │
│  Runs BEFORE Pydantic's type checking.                          │
│  You receive the RAW input (could be any type!).                │
│                                                                 │
│     Raw input → [YOUR CODE] → [Type coercion] → [Constraints]  │
│                                                                 │
│  You get: whatever the user sent (str, int, dict, anything).    │
│  Use when: You need to TRANSFORM data before type checking.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Example — mode="before" for data transformation:**

```python
from pydantic import BaseModel, field_validator

class Product(BaseModel):
    tags: list[str]
    
    @field_validator("tags", mode="before")
    @classmethod
    def split_tags_if_string(cls, v):
        """Accept both a list and a comma-separated string"""
        if isinstance(v, str):
            return [tag.strip() for tag in v.split(",")]
        return v
```

```python
# Both work now:
p1 = Product(tags=["python", "backend"])           # List ✅
p2 = Product(tags="python, backend, async")        # String → split ✅

print(p1.tags)  # ['python', 'backend']
print(p2.tags)  # ['python', 'backend', 'async']
```

**Example — mode="after" (default) for extra validation:**

```python
class Product(BaseModel):
    name: str
    price: float = Field(gt=0)
    sale_price: float | None = None
    
    @field_validator("sale_price")
    @classmethod
    def sale_price_must_be_positive(cls, v: float | None) -> float | None:
        # v is already validated as float or None
        if v is not None and v <= 0:
            raise ValueError("sale_price must be positive if provided")
        return v
```

**Gatekeeper analogy:**

> "mode='before' is like the gatekeeper reading a handwritten note and typing it into the system BEFORE running the standard checks. mode='after' is the gatekeeper doing a final review AFTER all the standard checks have passed."

---

## 4.3 @model_validator — Cross-Field Logic

**Sometimes validation depends on MULTIPLE fields together.**

```python
from pydantic import BaseModel, Field, model_validator

class DateRange(BaseModel):
    start_date: str
    end_date: str
    
    @model_validator(mode="after")
    def end_must_be_after_start(self) -> "DateRange":
        if self.end_date <= self.start_date:
            raise ValueError("end_date must be after start_date")
        return self
```

**Notice the difference from @field_validator:**

```
┌─────────────────────────────────────────────────────────────────┐
│           @field_validator vs @model_validator                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @field_validator:                                              │
│  ─────────────────                                              │
│  • Validates ONE field at a time                                │
│  • Decorated with @classmethod                                  │
│  • Receives: cls, field_value                                   │
│  • Returns: the field value                                     │
│  • Use for: single-field rules                                  │
│                                                                 │
│  @model_validator(mode="after"):                                │
│  ───────────────────────────────                                │
│  • Validates the WHOLE model                                    │
│  • Regular method (receives self)                               │
│  • Has access to ALL fields                                     │
│  • Returns: the model instance (self)                           │
│  • Use for: cross-field rules                                   │
│                                                                 │
│  @model_validator(mode="before"):                               │
│  ────────────────────────────────                               │
│  • Runs BEFORE any field validation                             │
│  • Decorated with @classmethod                                  │
│  • Receives: cls, raw data (dict)                               │
│  • Returns: the (possibly modified) dict                        │
│  • Use for: reshaping data before validation                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical example — password confirmation:**

```python
class UserRegistration(BaseModel):
    email: str
    password: str = Field(min_length=8)
    password_confirm: str
    
    @model_validator(mode="after")
    def passwords_must_match(self) -> "UserRegistration":
        if self.password != self.password_confirm:
            raise ValueError("passwords do not match")
        return self
```

**Practical example — conditional requirement:**

```python
class ShippingOrder(BaseModel):
    delivery_type: str  # "standard" or "express"
    phone_number: str | None = None
    
    @model_validator(mode="after")
    def express_requires_phone(self) -> "ShippingOrder":
        if self.delivery_type == "express" and not self.phone_number:
            raise ValueError(
                "phone_number is required for express delivery"
            )
        return self
```

```python
# ✅ Standard without phone — fine
order1 = ShippingOrder(delivery_type="standard")

# ❌ Express without phone — rejected!
order2 = ShippingOrder(delivery_type="express")
# ValidationError: phone_number is required for express delivery

# ✅ Express with phone — fine
order3 = ShippingOrder(delivery_type="express", phone_number="555-1234")
```

**The full validation order:**

```
┌─────────────────────────────────────────────────────────────────┐
│                FULL VALIDATION ORDER                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Raw Data Arrives                                               │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────────────┐                               │
│  │ @model_validator(mode="before")                              │
│  │ Runs on raw dict. Can reshape data.                          │
│  └──────────────┬───────────────┘                               │
│                 │                                               │
│                 ▼                                               │
│  ┌──────────────────────────────┐                               │
│  │ For EACH field:              │                               │
│  │  1. @field_validator(before) │  Transform raw value          │
│  │  2. Type coercion            │  "25" → 25                    │
│  │  3. Field constraints        │  gt=0, min_length=1           │
│  │  4. @field_validator(after)  │  Extra checks on valid value  │
│  └──────────────┬───────────────┘                               │
│                 │                                               │
│                 ▼                                               │
│  ┌──────────────────────────────┐                               │
│  │ @model_validator(mode="after")                               │
│  │ Runs on complete model. Cross-field checks.                  │
│  └──────────────┬───────────────┘                               │
│                 │                                               │
│                 ▼                                               │
│  Validated Model Instance ✅                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Common Mistakes with Validators

### Mistake 1: Forgetting to return the value

```python
# ❌ WRONG: Forgot to return — field becomes None!
@field_validator("name")
@classmethod
def validate_name(cls, v: str):
    if len(v) < 1:
        raise ValueError("name is empty")
    # Where's the return?? → field silently becomes None

# ✅ CORRECT: Always return the value
@field_validator("name")
@classmethod
def validate_name(cls, v: str) -> str:
    if len(v) < 1:
        raise ValueError("name is empty")
    return v  # ← Don't forget this!
```

---

### Mistake 2: Forgetting @classmethod

```python
# ❌ WRONG in Pydantic v2 — will raise a deprecation warning
@field_validator("name")
def validate_name(cls, v: str) -> str:
    return v.strip()

# ✅ CORRECT — always pair @field_validator with @classmethod
@field_validator("name")
@classmethod
def validate_name(cls, v: str) -> str:
    return v.strip()
```

---

### Mistake 3: Raising the wrong exception type

```python
# ❌ WRONG: TypeError is not caught by Pydantic
@field_validator("email")
@classmethod
def validate_email(cls, v: str) -> str:
    if "@" not in v:
        raise TypeError("invalid email")  # Pydantic doesn't catch this!
    return v

# ✅ CORRECT: Use ValueError (or AssertionError)
@field_validator("email")
@classmethod
def validate_email(cls, v: str) -> str:
    if "@" not in v:
        raise ValueError("email must contain @")  # Pydantic catches this
    return v
```

**Connection to error handling (Lecture 2, Week 1):**

> "Remember exception hierarchies from Week 1? Pydantic listens for `ValueError` and `AssertionError`. If you raise anything else, it won't be caught — it'll crash your app instead of producing a clean validation error."
---
## 4.5 Annotated Validators — Reusable Validation

**The limitation of @field_validator: it's tied to its model.**

```python
# You need username validation in three models. With @field_validator:
class UserCreate(BaseModel):
    username: str
    @field_validator("username")
    @classmethod
    def check_username(cls, v: str) -> str:
        if not v.isalnum(): raise ValueError("must be alphanumeric")
        return v.lower()

class AdminCreate(BaseModel):
    username: str
    @field_validator("username")      # ← Copy-pasted. Identical logic.
    @classmethod
    def check_username(cls, v: str) -> str:
        if not v.isalnum(): raise ValueError("must be alphanumeric")
        return v.lower()

# Duplicated. When the rule changes, you must update every model.
```

**The solution: extract the validation into a reusable type.**

```python
from typing import Annotated
from pydantic import BaseModel, AfterValidator, BeforeValidator

# Write the logic ONCE as a plain function
def validate_username(v: str) -> str:
    if not v.isalnum():
        raise ValueError("must contain only letters and numbers")
    return v.lower()        # ← also normalises to lowercase

# Create a TYPE ALIAS — this is now a type you can use anywhere
Username = Annotated[str, AfterValidator(validate_username)]
```

```python
# Use it in any model — zero duplication
class UserCreate(BaseModel):
    username: Username
    email: str

class AdminCreate(BaseModel):
    username: Username    # Same validation, different model
    role: str

class ApiKeyCreate(BaseModel):
    owner_username: Username   # Works on any field name
```

**Demonstration:**

```python
user = UserCreate(username="Alice42", email="a@b.com")
print(user.username)   # alice42  ← lowercased by the validator

# ❌ Invalid in any model that uses Username
AdminCreate(username="admin!!", role="superuser")
# ValidationError: must contain only letters and numbers
```

**The anatomy:**

```
┌─────────────────────────────────────────────────────────────────┐
│               ANNOTATED VALIDATOR ANATOMY                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Username = Annotated[str, AfterValidator(validate_username)]   │
│             │         │   │                │                    │
│             │         │   │                └─ Your function     │
│             │         │   └─ WHEN: after Pydantic type check    │
│             │         └─ BASE TYPE: must be a str               │
│             └─ "str WITH extra metadata attached"               │
│                                                                 │
│  AfterValidator   runs AFTER Pydantic coercion + type check     │
│                   receives: the already-typed value             │
│                   use for: extra checks on valid data           │
│                                                                 │
│  BeforeValidator  runs BEFORE Pydantic type check               │
│                   receives: the raw, untyped input              │
│                   use for: transforming input before typing     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Combining constraints and multiple validators in one type:**

```python
from pydantic import Field

def normalise_email(v: str) -> str:
    return v.strip().lower()

def reject_plus_addressing(v: str) -> str:
    local_part = v.split("@")[0]
    if "+" in local_part:
        raise ValueError("plus-addressing is not allowed")
    return v

# Stack Field constraints AND validators in one Annotated type
StrictEmail = Annotated[
    str,
    Field(min_length=5, max_length=254),   # ← length constraint
    BeforeValidator(normalise_email),       # ← runs first, normalises
    AfterValidator(reject_plus_addressing), # ← runs after typing
]

class NewsletterSignup(BaseModel):
    email: StrictEmail

# ✅ Normalised — whitespace and case stripped
signup = NewsletterSignup(email="  ALICE@EXAMPLE.COM  ")
print(signup.email)   # alice@example.com

# ❌ Plus-addressing rejected
# NewsletterSignup(email="alice+spam@example.com")
# ValidationError: plus-addressing is not allowed
```

**Execution order when validators are stacked:**

```
┌─────────────────────────────────────────────────────────────────┐
│      ANNOTATED EXECUTION ORDER (left to right in Annotated)     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Annotated[str, Field(min_length=5),                            │
│            BeforeValidator(f1), AfterValidator(f2)]             │
│                                                                 │
│  Raw input (any type)                                           │
│       │                                                         │
│       ▼                                                         │
│  BeforeValidator(f1)   ← receives raw input, returns raw value  │
│       │                                                         │
│       ▼                                                         │
│  Type coercion to str  ← Pydantic checks / coerces the type     │
│       │                                                         │
│       ▼                                                         │
│  Field(min_length=5)   ← built-in constraint checked here       │
│       │                                                         │
│       ▼                                                         │
│  AfterValidator(f2)    ← receives validated str, can raise      │
│       │                                                         │
│       ▼                                                         │
│  Final value ✅                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When to choose Annotated validators vs @field_validator:**

```
┌─────────────────────────────────────────────────────────────────┐
│      Annotated validators  vs  @field_validator                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Annotated validators:                                          │
│  ✅ Fully reusable — define once, use in any model              │
│  ✅ Composable — stack multiple in one type                     │
│  ✅ Works with TypeAdapter and standalone validation             │
│  ❌ Operates on one value only — no access to other fields      │
│                                                                 │
│  @field_validator:                                              │
│  ✅ Can read other fields via info.data (for context)           │
│  ✅ Custom logic that differs per model for the same type       │
│  ❌ Tied to the model — cannot be shared without copy-paste     │
│                                                                 │
│  Rule of thumb:                                                 │
│  Same validation logic in multiple models → Annotated type      │
│  Logic needs access to other fields on this specific model      │
│  → @field_validator or @model_validator                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 4.5                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Create a reusable SlugStr Annotated type that:                 │
│  1. Lowercases input and replaces spaces with hyphens           │
│     (BeforeValidator — transforms before type check)            │
│  2. After coercion, validates that the result only contains     │
│     lowercase letters, digits, and hyphens (AfterValidator)     │
│  3. Enforces max_length=100 (Field constraint)                  │
│                                                                 │
│  Then use SlugStr as the type for the slug field in BOTH:       │
│  - BlogPost(title: str, slug: SlugStr)                          │
│  - Product(name: str, slug: SlugStr)                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

import re
from typing import Annotated
from pydantic import BaseModel, Field, BeforeValidator, AfterValidator

def slugify(v: str) -> str:
    """Transform before type check: lowercase + spaces to hyphens."""
    return v.lower().replace(" ", "-")

def validate_slug(v: str) -> str:
    """After type check: only letters, digits, hyphens allowed."""
    if not re.match(r"^[a-z0-9-]+$", v):
        raise ValueError(
            "slug may only contain lowercase letters, digits, and hyphens"
        )
    return v

SlugStr = Annotated[
    str,
    Field(max_length=100),
    BeforeValidator(slugify),
    AfterValidator(validate_slug),
]

class BlogPost(BaseModel):
    title: str
    slug: SlugStr

class Product(BaseModel):
    name: str
    slug: SlugStr

# ✅ Input is normalised by BeforeValidator
post = BlogPost(title="Hello World", slug="Hello World Tutorial")
print(post.slug)   # hello-world-tutorial

product = Product(name="My Widget", slug="My Widget 2025")
print(product.slug)   # my-widget-2025

# ❌ Invalid character remains after normalisation
try:
    BlogPost(title="Test", slug="hello!!world")
except Exception as e:
    print(e)
# ValidationError: slug may only contain lowercase letters, digits, and hyphens
```
---

# PART 5: CONFIGURATION & SERIALIZATION

## 5.1 Model Configuration (model_config)

**The gatekeeper follows POLICIES. You can change the policies.**

```python
from pydantic import BaseModel, ConfigDict

class Product(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,   # Auto-strip "  hello  " → "hello"
        str_min_length=1,            # All strings must be non-empty
    )
    
    name: str
    category: str
    description: str = ""  # Has a default, so str_min_length 
                           # only applies to explicitly provided values
```

```python
product = Product(
    name="  Widget  ",
    category="  electronics  "
)

print(product.name)      # "Widget" — whitespace stripped!
print(product.category)  # "electronics"
```

**Useful configuration options:**


```
┌─────────────────────────────────────────────────────────────────┐
│                  MODEL CONFIGURATION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ConfigDict Option          │  What It Does                     │
│  ───────────────────────────│─────────────────────────          │
│  str_strip_whitespace=True  │  Strip leading/trailing spaces    │
│  str_min_length=1           │  All strings non-empty by default │
│  str_to_lower=True          │  Lowercase all string fields      │
│  extra="forbid"             │  Reject unknown fields ❌          │
│  extra="ignore"             │  Silently drop unknown fields     │
│  extra="allow"              │  Keep unknown fields              │
│  frozen=True                │  Make model immutable             │
│  from_attributes=True       │  Read from objects (ORM mode)     │
│  populate_by_name=True      │  Accept both alias and field name │
│  strict=True                │  Disable ALL type coercion —      │
│                             │  "25" will NOT become int 25      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**strict=True — disabling coercion model-wide:**

```python
from pydantic import BaseModel, ConfigDict

class LaxModel(BaseModel):                  # default behaviour
    age: int

class StrictModel(BaseModel):
    model_config = ConfigDict(strict=True)
    age: int

# Lax mode (default): "25" coerces to 25
LaxModel(age="25")    # ✅  age=25

# Strict mode: "25" is a string. An int field must receive an int.
StrictModel(age="25")
# ValidationError: Input should be a valid integer [type=int_type,
#   input_value='25', input_type=str]

StrictModel(age=25)   # ✅  only exact types pass
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  LAX MODE vs STRICT MODE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Lax (default):                 │  Strict:                      │
│  ───────────────                │  ────────                     │
│  "25"  → int 25    ✅           │  "25"  → ❌ must be int       │
│  25.0  → int 25    ✅           │  25.0  → ❌ must be int       │
│  "3.14"→ float 3.14✅           │  "3.14"→ ❌ must be float     │
│  1     → bool True ✅           │  1     → ❌ must be bool      │
│                                 │                               │
│  Use lax for: public APIs that  │  Use strict for: internal     │
│  receive data from browsers,    │  services, parsing DB rows    │
│  forms, and varied clients.     │  where types must be exact.   │
│                                 │                               │
└─────────────────────────────────────────────────────────────────┘
```

> "Strict mode is also available per-field without changing the whole model. Pydantic provides `StrictInt`, `StrictStr`, `StrictFloat`, `StrictBool` as individual types. You'll see these listed in the Quick Reference Card at the end."


**extra="forbid" — strict about unknown fields:**

```python
class StrictProduct(BaseModel):
    model_config = ConfigDict(extra="forbid")
    
    name: str
    price: float

# ❌ Rejected — "color" is not a known field
product = StrictProduct(name="Widget", price=9.99, color="red")
# ValidationError: Extra inputs are not permitted
```

> "For API input models, `extra='forbid'` is a good practice. If the client sends a field you don't expect, it's probably a typo or a misunderstanding. Catching it early prevents silent bugs."

---

## 5.2 from_attributes (Reading from Objects)

**Sometimes your data isn't a dict — it's a Python object with attributes.**

```python
# Imagine a database row (you'll see SQLAlchemy later in the course)
class ProductRow:
    """Simulates a database ORM object"""
    def __init__(self, id: int, name: str, price: float):
        self.id = id
        self.name = name
        self.price = price
```

**Without from_attributes:**

```python
class ProductRead(BaseModel):
    id: int
    name: str
    price: float

# ❌ This fails — Pydantic expects a dict, not an object
db_row = ProductRow(id=1, name="Widget", price=9.99)
product = ProductRead.model_validate(db_row)
# ValidationError!
```

**With from_attributes:**

```python
class ProductRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    
    id: int
    name: str
    price: float

# ✅ Now it reads from the object's attributes
db_row = ProductRow(id=1, name="Widget", price=9.99)
product = ProductRead.model_validate(db_row)

print(product.name)   # Widget
print(product.price)  # 9.99
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  from_attributes                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT from_attributes:                                       │
│  Pydantic reads from: dict keys                                 │
│  data["name"]  →  model.name                                    │
│                                                                 │
│  WITH from_attributes=True:                                     │
│  Pydantic reads from: object attributes OR dict keys            │
│  data.name  →  model.name                                       │
│                                                                 │
│  WHY THIS MATTERS:                                              │
│  Database ORMs (SQLAlchemy) return objects, not dicts.           │
│  from_attributes lets Pydantic convert ORM objects directly.    │
│  You'll use this extensively when we add databases.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Aliases (External vs Internal Names)

**The outside world and your Python code don't always agree on names.**

```python
from pydantic import BaseModel, Field, ConfigDict

class Product(BaseModel):
    model_config = ConfigDict(populate_by_name=True)
    
    product_name: str = Field(alias="productName")
    unit_price: float = Field(alias="unitPrice")
    is_available: bool = Field(alias="isAvailable", default=True)
```

**Now the model accepts both names:**

```python
# External API sends camelCase (common in JavaScript)
p1 = Product(productName="Widget", unitPrice=9.99)

# Internal Python code uses snake_case (Pythonic)
p2 = Product(product_name="Widget", unit_price=9.99)

# Both produce the same result
print(p1.product_name)  # Widget
print(p2.product_name)  # Widget
```

```
┌─────────────────────────────────────────────────────────────────┐
│                     ALIASES                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  External World (JSON)        │  Internal Python Code           │
│  ─────────────────────        │  ──────────────────             │
│  "productName"                │  product_name                   │
│  "unitPrice"                  │  unit_price                     │
│  "isAvailable"                │  is_available                   │
│                               │                                 │
│  alias= maps between them.    │                                 │
│                               │                                 │
│  populate_by_name=True allows both names to work.               │
│  Without it, ONLY the alias is accepted as input.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This pattern is everywhere. JavaScript frontends send camelCase. Python backends use snake_case. Aliases let the two worlds communicate without either compromising their naming conventions."

---

## 5.4 Serialization Control (.model_dump())

**Getting data OUT of a model — you control what's included.**

```python
class Product(BaseModel):
    model_config = ConfigDict(populate_by_name=True)
    
    product_name: str = Field(alias="productName")
    price: float
    internal_cost: float       # Internal only — don't expose!
    description: str = ""
    stock: int = 0
```

**Basic serialization:**

```python
product = Product(
    product_name="Widget",
    price=29.99,
    internal_cost=12.50,
    description="A fine widget",
    stock=100
)

# Default: all fields, Python field names
product.model_dump()
# {
#     "product_name": "Widget",
#     "price": 29.99,
#     "internal_cost": 12.50,
#     "description": "A fine widget",
#     "stock": 100
# }
```

**Control what's included:**

```python
# EXCLUDE specific fields
product.model_dump(exclude={"internal_cost"})
# {
#     "product_name": "Widget",
#     "price": 29.99,
#     "description": "A fine widget",
#     "stock": 100
# }

# INCLUDE only specific fields
product.model_dump(include={"product_name", "price"})
# {
#     "product_name": "Widget",
#     "price": 29.99
# }

# Use ALIAS names in output (for JSON API responses)
product.model_dump(by_alias=True, exclude={"internal_cost"})
# {
#     "productName": "Widget",   ← camelCase alias used
#     "price": 29.99,
#     "description": "A fine widget",
#     "stock": 100
# }
```

**Smart exclusion options:**

```python
class UserProfile(BaseModel):
    name: str
    email: str
    bio: str | None = None
    website: str | None = None
    age: int | None = None

user = UserProfile(name="Alice", email="alice@example.com", bio="Hi!")

# EXCLUDE_NONE — Drop fields that are None
user.model_dump(exclude_none=True)
# {
#     "name": "Alice",
#     "email": "alice@example.com",
#     "bio": "Hi!"
# }
# "website" and "age" are gone — they were None

# EXCLUDE_UNSET — Drop fields that weren't explicitly set
user.model_dump(exclude_unset=True)
# {
#     "name": "Alice",
#     "email": "alice@example.com",
#     "bio": "Hi!"
# }
```

```
┌─────────────────────────────────────────────────────────────────┐
│               SERIALIZATION OPTIONS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Option            │  What It Does                              │
│  ──────────────────│──────────────────────────────              │
│  exclude={...}     │  Remove specific fields                    │
│  include={...}     │  Keep ONLY these fields                    │
│  by_alias=True     │  Use alias names in output                 │
│  exclude_none=True │  Drop fields where value is None           │
│  exclude_unset=True│  Drop fields that weren't provided         │
│                    │                                            │
│  .model_dump()      → Returns dict                              │
│  .model_dump_json() → Returns JSON string                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Gatekeeper analogy:**

> "Serialization is the gatekeeper working in REVERSE. When data goes OUT, the gatekeeper decides what information to share and what to keep internal. The visitor's full file stays in the building — only the relevant info goes out on the badge."
---
## 5.5 Computed Fields (@computed_field)

**The problem: you need a field that is always derived from other fields and should appear in your serialized output — but you don't want to store it separately.**

```python
# ❌ Without computed_field — you'd compute it every time manually
user = User(first_name="Alice", last_name="Smith")
full_name = f"{user.first_name} {user.last_name}"   # repeated everywhere
```

**`@computed_field` declares a derived field that appears in model_dump():**

```python
from pydantic import BaseModel, computed_field

class User(BaseModel):
    first_name: str
    last_name: str

    @computed_field
    @property
    def full_name(self) -> str:          # ← return type is REQUIRED
        return f"{self.first_name} {self.last_name}"
```

```python
user = User(first_name="Alice", last_name="Smith")

# Access like any attribute
print(user.full_name)          # Alice Smith

# Appears automatically in model_dump() and model_dump_json()
print(user.model_dump())
# {"first_name": "Alice", "last_name": "Smith", "full_name": "Alice Smith"}

# Cannot be set — it is always derived
# User(first_name="Alice", last_name="Smith", full_name="Bob")
# → Pydantic ignores the argument (computed fields are not settable)
```

**Rules for @computed_field:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  @computed_field RULES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. BOTH decorators are required:                               │
│       @computed_field     ← outer decorator (Pydantic aware)   │
│       @property           ← inner decorator (Python property)   │
│                                                                 │
│  2. Return type annotation is REQUIRED in Pydantic v2.          │
│     Without it: PydanticUserError at class definition time.     │
│                                                                 │
│  3. Computed fields ARE included in:                            │
│     ✅ model_dump()   ✅ model_dump_json()   ✅ OpenAPI schema  │
│                                                                 │
│  4. Computed fields are NOT settable during __init__.            │
│     Attempting to pass them as arguments is ignored.            │
│                                                                 │
│  5. They are recalculated on every access — they are            │
│     properties, not stored values.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical example — order with computed totals:**

```python
from pydantic import BaseModel, Field, computed_field

class OrderItem(BaseModel):
    product_name: str
    quantity: int = Field(gt=0)
    unit_price: float = Field(gt=0)

    @computed_field
    @property
    def subtotal(self) -> float:
        return self.quantity * self.unit_price

class Order(BaseModel):
    items: list[OrderItem]
    tax_rate: float = Field(default=0.08, ge=0, lt=1)   # 8% default

    @computed_field
    @property
    def subtotal(self) -> float:
        return sum(item.subtotal for item in self.items)

    @computed_field
    @property
    def tax(self) -> float:
        return round(self.subtotal * self.tax_rate, 2)

    @computed_field
    @property
    def total(self) -> float:
        return round(self.subtotal + self.tax, 2)
```

```python
order = Order(
    items=[
        OrderItem(product_name="Keyboard", quantity=1, unit_price=89.99),
        OrderItem(product_name="Mouse", quantity=2, unit_price=29.99),
    ]
)

print(order.subtotal)   # 149.97
print(order.tax)        # 12.0
print(order.total)      # 161.97

print(order.model_dump())
# {
#   "items": [
#     {"product_name": "Keyboard", "quantity": 1, "unit_price": 89.99,
#      "subtotal": 89.99},
#     {"product_name": "Mouse", "quantity": 2, "unit_price": 29.99,
#      "subtotal": 59.98}
#   ],
#   "tax_rate": 0.08,
#   "subtotal": 149.97,
#   "tax": 12.0,
#   "total": 161.97
# }
```

**How it connects to FastAPI:**

> "When you use a model with `@computed_field` as a `response_model=`, the computed fields appear in the Swagger UI schema and in the API response automatically. The client gets `full_name` even though you only store `first_name` and `last_name`."

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 5.5                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Build a Rectangle model with:                                  │
│  - width: float (positive)                                      │
│  - height: float (positive)                                     │
│  - area: computed (width × height)                              │
│  - perimeter: computed (2 × (width + height))                   │
│  - is_square: computed (True when width == height)              │
│                                                                 │
│  Verify that model_dump() includes all five fields.             │
│  What happens if you try to pass area=100 to the constructor?   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

from pydantic import BaseModel, Field, computed_field

class Rectangle(BaseModel):
    width: float = Field(gt=0)
    height: float = Field(gt=0)

    @computed_field
    @property
    def area(self) -> float:
        return self.width * self.height

    @computed_field
    @property
    def perimeter(self) -> float:
        return 2 * (self.width + self.height)

    @computed_field
    @property
    def is_square(self) -> bool:
        return self.width == self.height

r = Rectangle(width=4.0, height=6.0)
print(r.area)           # 24.0
print(r.perimeter)      # 20.0
print(r.is_square)      # False

print(r.model_dump())
# {"width": 4.0, "height": 6.0, "area": 24.0,
#  "perimeter": 20.0, "is_square": False}

# Trying to pass area=100 is silently IGNORED
r2 = Rectangle(width=4.0, height=6.0, area=100)
print(r2.area)          # 24.0  ← still 4×6, not 100
```

---

## 5.6 Field Serializer (@field_serializer)

**The problem: model_dump() uses Pydantic's default serialization for every type. Sometimes you need a specific field to serialize differently — a `datetime` formatted as a custom string, a `Decimal` as a string to preserve precision, or a sensitive field masked.**

> "Important: `@field_serializer` changes what model_dump() RETURNS for that field. It does NOT change the value stored in the model. The attribute still holds the original type — the serializer only fires when you serialize."

```python
from pydantic import BaseModel, field_serializer
from datetime import datetime

class Event(BaseModel):
    name: str
    event_time: datetime

    @field_serializer("event_time")
    def serialize_event_time(self, dt: datetime) -> str:
        return dt.strftime("%d %b %Y at %H:%M UTC")

event = Event(name="Deploy", event_time=datetime(2025, 6, 15, 14, 30))

# The attribute is still a datetime object in memory
print(event.event_time)               # 2025-06-15 14:30:00
print(type(event.event_time))         # <class 'datetime.datetime'>

# But model_dump() uses the serializer
print(event.model_dump())
# {"name": "Deploy", "event_time": "15 Jun 2025 at 14:30 UTC"}

print(event.model_dump_json())
# {"name":"Deploy","event_time":"15 Jun 2025 at 14:30 UTC"}
```

**The anatomy:**

```
┌─────────────────────────────────────────────────────────────────┐
│               @field_serializer ANATOMY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @field_serializer("event_time")                                │
│  def serialize_event_time(                                      │
│      self,             ← instance method (has access to self)  │
│      value: datetime,  ← the field's stored value              │
│  ) -> str:             ← return type = what goes into JSON     │
│      return value.strftime(...)                                 │
│                                                                 │
│  The method name is arbitrary.                                  │
│  You can serialise multiple fields with one decorator:          │
│  @field_serializer("field_a", "field_b")                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical example — masking sensitive data:**

```python
from pydantic import BaseModel, field_serializer

class UserWithPassword(BaseModel):
    username: str
    password_hash: str     # The real hash is stored in memory

    @field_serializer("password_hash")
    def mask_password(self, value: str) -> str:
        return "***"       # Never expose the hash in output

user = UserWithPassword(username="alice", password_hash="$argon2id$v=19$...")

# The full hash IS accessible from Python code (for DB storage)
print(user.password_hash)          # $argon2id$v=19$...

# But model_dump() masks it
print(user.model_dump())
# {"username": "alice", "password_hash": "***"}
```

**Practical example — Decimal serialization:**

```python
from decimal import Decimal
from pydantic import BaseModel, field_serializer

class FinancialRecord(BaseModel):
    amount: Decimal
    currency: str = "USD"

    @field_serializer("amount")
    def serialize_amount(self, value: Decimal) -> str:
        # Preserve full decimal precision as a string
        return str(value)

record = FinancialRecord(amount=Decimal("9.99"))

print(record.model_dump())
# {"amount": "9.99", "currency": "USD"}
# ← String, not float — no floating-point precision loss
```

**Connecting to what you learned in section 5.4:**

> "In section 5.4, you learned `model_dump(exclude={'internal_cost'})` to control output at the call site. `@field_serializer` controls output at the definition site — the transformation is baked into the model itself. Use `exclude=` for situational omission; use `@field_serializer` for transformation that should always apply everywhere."

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 5.6                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Build a LogEntry model with:                                   │
│  - message: str                                                 │
│  - level: str (e.g., "INFO", "ERROR")                           │
│  - timestamp: datetime                                          │
│                                                                 │
│  Add a @field_serializer for timestamp that outputs it in       │
│  ISO 8601 format with a trailing "Z"                            │
│  (e.g., "2025-06-15T14:30:00Z").                                │
│                                                                 │
│  After building an instance, confirm that:                      │
│  1. log.timestamp is still a datetime object in memory.         │
│  2. log.model_dump()["timestamp"] is the formatted string.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

from pydantic import BaseModel, field_serializer
from datetime import datetime, timezone

class LogEntry(BaseModel):
    message: str
    level: str
    timestamp: datetime

    @field_serializer("timestamp")
    def serialize_timestamp(self, dt: datetime) -> str:
        # Convert to UTC, format as ISO 8601 with Z suffix
        utc_dt = dt.astimezone(timezone.utc)
        return utc_dt.strftime("%Y-%m-%dT%H:%M:%SZ")

log = LogEntry(
    message="Service started",
    level="INFO",
    timestamp=datetime(2025, 6, 15, 14, 30, 0, tzinfo=timezone.utc)
)

# 1. Still a datetime in memory
print(log.timestamp)               # 2025-06-15 14:30:00+00:00
print(type(log.timestamp))         # <class 'datetime.datetime'>

# 2. Serialised as formatted string
data = log.model_dump()
print(data["timestamp"])           # 2025-06-15T14:30:00Z
print(type(data["timestamp"]))     # <class 'str'>

print(log.model_dump_json())
# {"message":"Service started","level":"INFO","timestamp":"2025-06-15T14:30:00Z"}
```

---

## 5.7 model_post_init — Post-Initialization Hook

**Connection to what you've learned:**

> "In Week 1, Lecture 2, you saw `__post_init__` in dataclasses — a hook that runs after the constructor assigns all fields. Pydantic has the equivalent: `model_post_init`. It runs after all fields have been validated and set."

```python
from pydantic import BaseModel

class Order(BaseModel):
    items: list[str]
    quantities: list[int]

    def model_post_init(self, __context) -> None:
        """Runs after validation. All fields are already validated."""
        if len(self.items) != len(self.quantities):
            raise ValueError(
                f"items ({len(self.items)}) and quantities "
                f"({len(self.quantities)}) must have the same length"
            )
```

```python
# ✅ Lengths match
order = Order(items=["Keyboard", "Mouse"], quantities=[1, 2])

# ❌ Lengths differ — caught in model_post_init
Order(items=["Keyboard"], quantities=[1, 2])
# ValueError: items (1) and quantities (2) must have the same length
```

> "Wait — why not use `@model_validator(mode='after')` for this? You absolutely could. `model_post_init` is the lower-level hook. It's best used for initialization side effects — setting private attributes, establishing connections, or computing stored values. For pure cross-field validation, prefer `@model_validator`."

**The key distinction — stored vs derived:**

```
┌─────────────────────────────────────────────────────────────────┐
│         model_post_init  vs  @computed_field                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  @computed_field                                                │
│  ─────────────                                                  │
│  Derived. Recalculated on every access.                         │
│  Does NOT store the value — it's always re-run.                 │
│  Use when the derived value is cheap to compute.                │
│  Appears in model_dump() automatically.                         │
│                                                                 │
│  model_post_init                                                │
│  ────────────────                                               │
│  Runs once at creation. CAN store a value in a field.           │
│  Use when the derived value is expensive and should be cached.  │
│  Use for side effects (DB connections, logging, etc.).          │
│  Setting a regular field here DOES store it.                    │
│                                                                 │
│  Rule of thumb:                                                 │
│  Simple derived field → @computed_field                         │
│  Expensive computation or side effects → model_post_init        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical example — storing a pre-computed hash:**

```python
import hashlib
from pydantic import BaseModel, Field

class CachedDocument(BaseModel):
    content: str
    content_hash: str = ""   # Stored, not derived — computed once

    def model_post_init(self, __context) -> None:
        # Compute once on creation, stored in the field
        self.content_hash = hashlib.sha256(
            self.content.encode()
        ).hexdigest()
```

```python
doc = CachedDocument(content="Hello, World!")
print(doc.content_hash)   # 315f5bdb76d078c43b8ac0064e4a0164...
print(doc.model_dump())
# {"content": "Hello, World!", "content_hash": "315f5bdb..."}
```

**The signature — `__context` explained:**

```python
def model_post_init(self, __context) -> None:
    ...
```

> "The `__context` parameter carries an optional validation context object. For normal model creation — `MyModel(...)` — it is `None`. It only has a value when you call `MyModel.model_validate(data, context={...})` and pass a context dict. You will see this used in advanced dependency injection patterns. For now, just accept the parameter and ignore it."

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 5.7                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Build a FileUpload model with:                                 │
│  - filename: str                                                │
│  - content: bytes                                               │
│  - size_bytes: int = 0  (stored)                                │
│  - extension: str = ""  (stored)                                │
│                                                                 │
│  In model_post_init:                                            │
│  - Set size_bytes = len(content)                                │
│  - Set extension = the file extension from filename             │
│    (e.g., "report.pdf" → ".pdf"), empty string if none.        │
│                                                                 │
│  These should NOT be @computed_field because they are           │
│  expensive to recompute and are stored for direct DB write.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

import os
from pydantic import BaseModel

class FileUpload(BaseModel):
    filename: str
    content: bytes
    size_bytes: int = 0
    extension: str = ""

    def model_post_init(self, __context) -> None:
        self.size_bytes = len(self.content)
        _, ext = os.path.splitext(self.filename)
        self.extension = ext.lower()   # ".pdf", ".png", "" etc.

upload = FileUpload(filename="report.pdf", content=b"PDF content here")
print(upload.size_bytes)    # 16
print(upload.extension)     # .pdf

# Both are stored and appear in model_dump
print(upload.model_dump())
# {"filename": "report.pdf", "content": b"PDF content here",
#  "size_bytes": 16, "extension": ".pdf"}
```

---

## 5.8 Private Attributes (PrivateAttr)

**The problem: you need to store internal state — a database session, a cached connection, a call counter — that should never appear in your schema, your docs, or your serialized output.**

> "A regular field with `exclude={'_internal'}` in every model_dump() call is fragile. Private attributes are the clean solution: they are completely invisible to Pydantic's schema, validation, and serialization."

```python
from pydantic import BaseModel, PrivateAttr

class ApiClient(BaseModel):
    base_url: str
    timeout: int = 30
    _request_count: int = PrivateAttr(default=0)     # not in schema!
    _session: object = PrivateAttr(default=None)      # not in schema!

    def get(self, path: str) -> str:
        self._request_count += 1
        return f"GET {self.base_url}{path}"
```

```python
client = ApiClient(base_url="https://api.example.com")

client.get("/users")
client.get("/products")

print(client._request_count)   # 2  ← accessible from code
print(client.base_url)         # https://api.example.com

# model_dump() shows ONLY regular fields
print(client.model_dump())
# {"base_url": "https://api.example.com", "timeout": 30}
# _request_count and _session are GONE
```

**Rules:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   PrivateAttr RULES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Name MUST start with an underscore: _name                   │
│                                                                 │
│  2. Not included in:                                            │
│     ❌ model_dump()         ❌ model_dump_json()                │
│     ❌ model_json_schema()  ❌ model_fields                     │
│     ❌ __init__ signature   (you cannot pass them to Model(...))│
│                                                                 │
│  3. Not validated by Pydantic — any value can be assigned.      │
│                                                                 │
│  4. Initialized from PrivateAttr(default=...) or                │
│     PrivateAttr(default_factory=...) — same API as Field().     │
│                                                                 │
│  5. Initialized with default BEFORE model_post_init runs.        │
│     Override in model_post_init if the value depends on fields. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Initializing a private attribute from field values — combine with model_post_init:**

```python
from pydantic import BaseModel, PrivateAttr
from typing import Any

class CachedUser(BaseModel):
    user_id: int
    display_name: str
    _cache: dict[str, Any] = PrivateAttr(default_factory=dict)
    _cache_key: str = PrivateAttr(default="")

    def model_post_init(self, __context) -> None:
        # Private attribute value derived from a field
        self._cache_key = f"user:{self.user_id}"

    def cache_set(self, key: str, value: Any) -> None:
        self._cache[f"{self._cache_key}:{key}"] = value

    def cache_get(self, key: str) -> Any | None:
        return self._cache.get(f"{self._cache_key}:{key}")
```

```python
user = CachedUser(user_id=42, display_name="Alice")
user.cache_set("permissions", ["read", "write"])

print(user.cache_get("permissions"))    # ["read", "write"]
print(user._cache_key)                  # user:42

# None of the private state appears in the output
print(user.model_dump())
# {"user_id": 42, "display_name": "Alice"}
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 5.8                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Build a RateLimitedClient model:                               │
│  - base_url: str (regular field)                                │
│  - max_calls: int = 10 (regular field)                          │
│  - _calls_made: int (PrivateAttr, default 0)                    │
│                                                                 │
│  Add a method call(path: str) -> str that:                      │
│  - Increments _calls_made                                       │
│  - Raises ValueError if _calls_made > max_calls                 │
│  - Otherwise returns f"GET {base_url}{path}"                    │
│                                                                 │
│  Confirm that model_dump() shows only base_url and max_calls.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

from pydantic import BaseModel, PrivateAttr

class RateLimitedClient(BaseModel):
    base_url: str
    max_calls: int = 10
    _calls_made: int = PrivateAttr(default=0)

    def call(self, path: str) -> str:
        self._calls_made += 1
        if self._calls_made > self.max_calls:
            raise ValueError(
                f"Rate limit exceeded: {self._calls_made} / {self.max_calls}"
            )
        return f"GET {self.base_url}{path}"

client = RateLimitedClient(base_url="https://api.example.com", max_calls=3)
print(client.call("/users"))     # GET https://api.example.com/users
print(client.call("/items"))     # GET https://api.example.com/items
print(client.call("/orders"))    # GET https://api.example.com/orders

try:
    client.call("/extra")        # 4th call — over the limit
except ValueError as e:
    print(e)                     # Rate limit exceeded: 4 / 3

# _calls_made is completely invisible to model_dump
print(client.model_dump())
# {"base_url": "https://api.example.com", "max_calls": 3}
```

---

## 5.9 model_copy() — Creating Modified Copies

**The problem: you have a validated model and want a modified version — but you don't want to reconstruct it from scratch.**

```python
product = Product(name="Widget", price=9.99, stock=100)

# The hard way — rebuild manually:
discounted = Product(
    name=product.name,        # copy every field
    price=7.99,               # change one
    stock=product.stock,      # copy the rest
    # ...  tedious, error-prone, misses future fields
)
```

**model_copy() is the clean solution:**

```python
product = Product(name="Widget", price=9.99, stock=100)

# Create a copy with only price changed
discounted = product.model_copy(update={"price": 7.99})

print(discounted.price)   # 7.99
print(discounted.name)    # Widget      ← copied from original
print(discounted.stock)   # 100         ← copied from original
print(product.price)      # 9.99        ← original is UNCHANGED
```

**Deep copy — for models with mutable nested structures:**

```python
import copy
from pydantic import BaseModel

class Cart(BaseModel):
    user_id: int
    items: list[str]

cart = Cart(user_id=1, items=["keyboard", "mouse"])

# Shallow copy — items list is SHARED between original and copy
shallow = cart.model_copy()
shallow.items.append("monitor")      # ← modifies the ORIGINAL too!
print(cart.items)                    # ['keyboard', 'mouse', 'monitor'] ← oops

# Deep copy — creates fully independent nested objects
cart2 = Cart(user_id=1, items=["keyboard", "mouse"])
deep = cart2.model_copy(deep=True)
deep.items.append("monitor")         # ← independent from original
print(cart2.items)                   # ['keyboard', 'mouse'] ← unchanged
```

**CRITICAL: model_copy(update=...) does NOT validate the update:**

```
┌─────────────────────────────────────────────────────────────────┐
│               model_copy() DOES NOT VALIDATE UPDATES            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  product = Product(name="Widget", price=9.99)                   │
│                                                                 │
│  # ❌ This SUCCEEDS — even though price=-1 is invalid!           │
│  bad = product.model_copy(update={"price": -1})                 │
│  print(bad.price)   # -1  ← validation was skipped!             │
│                                                                 │
│  model_copy() is designed for fast, trusted internal copies.    │
│  It skips validation deliberately.                              │
│                                                                 │
│  ✅ For a validated copy:                                        │
│  new_data = product.model_dump()                                │
│  new_data["price"] = -1                                         │
│  validated = Product.model_validate(new_data)  ← validates!    │
│  # ValidationError: Input should be greater than 0              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**model_copy with frozen=True models:**

```python
from pydantic import BaseModel, ConfigDict

class ImmutableConfig(BaseModel):
    model_config = ConfigDict(frozen=True)
    host: str
    port: int = 5432

config = ImmutableConfig(host="localhost")

# ❌ Cannot mutate a frozen model
# config.port = 5433   → ValidationError: Instance is frozen

# ✅ model_copy is the ONLY way to get a "modified" version
dev_config = config.model_copy(update={"host": "dev.db.example.com"})
print(dev_config.host)    # dev.db.example.com
print(config.host)        # localhost  ← original unchanged
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 5.9                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Given:                                                         │
│  class ServerConfig(BaseModel):                                 │
│      model_config = ConfigDict(frozen=True)                     │
│      host: str                                                  │
│      port: int = Field(gt=0, le=65535)                          │
│      tls: bool = False                                          │
│                                                                 │
│  prod = ServerConfig(host="prod.example.com", port=443, tls=True│
│                                                                 │
│  1. Create a staging copy that uses host="staging.example.com"  │
│     and port=8443, keeping tls=True from prod.                  │
│                                                                 │
│  2. Try model_copy(update={"port": 99999}).                     │
│     Does it raise a ValidationError? Why or why not?            │
│     How would you get a VALIDATED copy?                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

from pydantic import BaseModel, ConfigDict, Field

class ServerConfig(BaseModel):
    model_config = ConfigDict(frozen=True)
    host: str
    port: int = Field(gt=0, le=65535)
    tls: bool = False

prod = ServerConfig(host="prod.example.com", port=443, tls=True)

# 1. Create staging copy with model_copy — only change what differs
staging = prod.model_copy(update={
    "host": "staging.example.com",
    "port": 8443,
})
print(staging.host)   # staging.example.com
print(staging.port)   # 8443
print(staging.tls)    # True  ← copied from prod, unchanged

# 2. model_copy does NOT validate — port=99999 succeeds!
bad = prod.model_copy(update={"port": 99999})
print(bad.port)       # 99999 ← validation was bypassed!

# For a validated copy, rebuild via model_dump + model_validate
new_data = prod.model_dump()
new_data["port"] = 99999
ServerConfig.model_validate(new_data)
# ValidationError: port — Input should be less than or equal to 65535
```

---

# PART 6: REQUEST VS RESPONSE MODELS

## 6.1 Why One Model Isn't Enough

**Ask the class:**

> "You have a User model with fields: id, name, email, password_hash, created_at. When a user REGISTERS, should they send all of these? When you RETURN user data, should you include the password_hash?"

```
┌─────────────────────────────────────────────────────────────────┐
│              THE ONE-MODEL PROBLEM                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  If you use ONE model for everything:                           │
│                                                                 │
│  class User(BaseModel):                                         │
│      id: int                ← Who generates this? DB does.      │
│      name: str                                                  │
│      email: str                                                 │
│      password_hash: str     ← NEVER send this to the client!    │
│      created_at: datetime   ← Who generates this? Server does.  │
│                                                                 │
│  CREATE: Client sends {name, email, password}                   │
│          They DON'T have id or created_at yet.                  │
│          They send password, NOT password_hash.                  │
│                                                                 │
│  READ:   Server returns {id, name, email, created_at}           │
│          They MUST NOT see password_hash.                        │
│          They SHOULD see id and created_at.                      │
│                                                                 │
│  UPDATE: Client sends {name?, email?} (all optional)            │
│          They shouldn't change id or created_at.                 │
│                                                                 │
│  Same entity. Three COMPLETELY DIFFERENT shapes.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 The Create / Read / Update Pattern

**The industry-standard solution: separate models for separate purposes.**

```python
from pydantic import BaseModel, Field, ConfigDict
from datetime import datetime

# SHARED FIELDS — the common foundation
class ProductBase(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    price: float = Field(gt=0)
    category: str
    description: str = ""


# CREATE — what the client sends to make a new product
class ProductCreate(ProductBase):
    stock: int = Field(default=0, ge=0)
    # No id (DB generates it)
    # No created_at (server generates it)


# READ — what the server returns to the client
class ProductRead(ProductBase):
    model_config = ConfigDict(from_attributes=True)
    
    id: int                       # ← DB generated
    stock: int
    created_at: datetime          # ← Server generated


# UPDATE — what the client sends to modify a product
class ProductUpdate(BaseModel):
    name: str | None = None       # ← All fields optional!
    price: float | None = Field(default=None, gt=0)
    category: str | None = None
    description: str | None = None
    stock: int | None = Field(default=None, ge=0)
```

**Visualize the inheritance:**

```
┌─────────────────────────────────────────────────────────────────┐
│              MODEL INHERITANCE PATTERN                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ProductBase                                   │
│                    ├─ name: str                                  │
│                    ├─ price: float                               │
│                    ├─ category: str                              │
│                    └─ description: str = ""                      │
│                         │                                       │
│               ┌─────────┴──────────┐                            │
│               │                    │                            │
│         ProductCreate         ProductRead                       │
│         ├─ stock: int = 0     ├─ id: int                        │
│         │                     ├─ stock: int                     │
│         │                     ├─ created_at: datetime            │
│         │  (no id, no date)   │  (from_attributes=True)         │
│         │                     │                                 │
│         │  Client → Server    │  Server → Client                │
│                                                                 │
│                                                                 │
│         ProductUpdate (standalone — all optional)                │
│         ├─ name: str | None                                     │
│         ├─ price: float | None                                  │
│         ├─ category: str | None                                 │
│         ├─ description: str | None                              │
│         └─ stock: int | None                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "ProductUpdate doesn't inherit from ProductBase because every field must be optional. If it inherited, name would be required — but you might want to update only the price. A standalone model with all-optional fields is the cleanest approach."

---

## 6.3 Inheritance for Shared Fields

**Why inherit? Because DRY — Don't Repeat Yourself.**

```python
# ❌ WITHOUT inheritance — repeated fields everywhere
class ProductCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    price: float = Field(gt=0)
    category: str
    description: str = ""

class ProductRead(BaseModel):
    name: str = Field(min_length=1, max_length=100)  # Same!
    price: float = Field(gt=0)                       # Same!
    category: str                                     # Same!
    description: str = ""                             # Same!
    id: int
    created_at: datetime

# If "name" max_length changes to 200, you must update BOTH models.
# You WILL forget one. Guaranteed.
```

```python
# ✅ WITH inheritance — shared fields defined ONCE
class ProductBase(BaseModel):
    name: str = Field(min_length=1, max_length=100)  # Only here!
    price: float = Field(gt=0)                       # Only here!
    category: str                                     # Only here!
    description: str = ""                             # Only here!

class ProductCreate(ProductBase):
    stock: int = Field(default=0, ge=0)

class ProductRead(ProductBase):
    model_config = ConfigDict(from_attributes=True)
    id: int
    stock: int
    created_at: datetime
```

> "Change `max_length=100` to `max_length=200` in one place — ProductBase — and both Create and Read models pick it up automatically."

---

## 6.4 Putting It Together with FastAPI

**Connection to what you've learned (Lecture 2, this week):**

> "In Lecture 2, you learned FastAPI basics — path operations, request bodies. Now you see how Pydantic models plug into FastAPI endpoints. The model IS the API contract."

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

# Simulated storage (in-memory for now)
products: dict[int, dict] = {}
next_id = 1

@app.post("/products", response_model=ProductRead, status_code=201)
async def create_product(product: ProductCreate):
    global next_id
    from datetime import datetime, timezone
    
    product_data = product.model_dump()
    product_data["id"] = next_id
    product_data["created_at"] = datetime.now(timezone.utc)
    
    products[next_id] = product_data
    next_id += 1
    
    return product_data  # FastAPI uses ProductRead to filter response


@app.get("/products/{product_id}", response_model=ProductRead)
async def get_product(product_id: int):
    if product_id not in products:
        raise HTTPException(404, f"Product {product_id} not found")
    return products[product_id]


@app.patch("/products/{product_id}", response_model=ProductRead)
async def update_product(product_id: int, updates: ProductUpdate):
    if product_id not in products:
        raise HTTPException(404, f"Product {product_id} not found")
    
    # Only update fields that were explicitly provided
    update_data = updates.model_dump(exclude_unset=True)
    products[product_id].update(update_data)
    
    return products[product_id]
```

**The data flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  DATA FLOW WITH MODELS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  POST /products                                                 │
│                                                                 │
│  Client sends JSON ──▶ ProductCreate validates ──▶ Your code    │
│                        (rejects bad data)         (safe data)   │
│                                                                 │
│  Your code ──▶ ProductRead filters ──▶ Client receives JSON     │
│  (all data)   (hides internal_cost,   (only what they should    │
│                adds id, created_at)    see)                     │
│                                                                 │
│                                                                 │
│  PATCH /products/{id}                                           │
│                                                                 │
│  Client sends JSON ──▶ ProductUpdate validates ──▶ Your code    │
│  (partial fields)      (all optional, still      (merge with    │
│                         validates provided ones)  existing data)│
│                                                                 │
│  model_dump(exclude_unset=True) ensures you only                │
│  overwrite fields the client actually sent.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What `response_model=ProductRead` does:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Your endpoint returns:         FastAPI + ProductRead filters:  │
│  {                              {                               │
│    "id": 1,                       "id": 1,                     │
│    "name": "Widget",              "name": "Widget",            │
│    "price": 29.99,                "price": 29.99,              │
│    "category": "electronics",     "category": "electronics",   │
│    "description": "",             "description": "",           │
│    "stock": 50,                   "stock": 50,                 │
│    "created_at": "2025-...",      "created_at": "2025-..."     │
│    "internal_cost": 12.50  ← ❌ │  }                           │
│  }                                                              │
│                              internal_cost REMOVED by           │
│                              response_model automatically!      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This is why response_model exists. Even if your code accidentally includes sensitive data, FastAPI uses the Pydantic model to STRIP it from the response. Defense in depth."
---
---

## 6.5 response_model Advanced Filtering

**Tech debt from W3L2 — paying off: `response_model_include`, `response_model_exclude`, `response_model_by_alias`.**

**You already know `response_model=ProductRead` from section 6.4. FastAPI's route decorators support additional parameters that give you fine-grained control over the response shape — without modifying the model or changing your endpoint logic.**

```python
from fastapi import FastAPI

app = FastAPI()

class ProductRead(BaseModel):
    model_config = ConfigDict(populate_by_name=True)
    id: int
    product_name: str = Field(alias="productName")
    price: float
    internal_cost: float
    stock: int
    created_at: datetime
```

**response_model_include — return ONLY specified fields:**

```python
# Only id, product_name, and price are included in the response
@app.get(
    "/products/{product_id}/public",
    response_model=ProductRead,
    response_model_include={"id", "product_name", "price"},
)
async def get_product_public(product_id: int):
    return products[product_id]   # endpoint returns all data
    # FastAPI filters it down to only id, product_name, price
```

**response_model_exclude — return everything EXCEPT specified fields:**

```python
# All fields except internal_cost are in the response
@app.get(
    "/products/{product_id}",
    response_model=ProductRead,
    response_model_exclude={"internal_cost"},
)
async def get_product(product_id: int):
    return products[product_id]
```

**response_model_by_alias — use alias names in the JSON response:**

```python
# Response JSON uses "productName" (the alias) not "product_name" (the field name)
@app.get(
    "/products/{product_id}",
    response_model=ProductRead,
    response_model_by_alias=True,
    response_model_exclude={"internal_cost"},
)
async def get_product_aliased(product_id: int):
    return products[product_id]
    # Client receives: {"id": 1, "productName": "Widget", ...}
```

**How these map to model_dump():**

```
┌─────────────────────────────────────────────────────────────────┐
│          response_model PARAMS vs model_dump() PARAMS           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Route decorator param          │  Equivalent in code           │
│  ───────────────────────────────│─────────────────────────────  │
│  response_model_include={...}   │  .model_dump(include={...})   │
│  response_model_exclude={...}   │  .model_dump(exclude={...})   │
│  response_model_by_alias=True   │  .model_dump(by_alias=True)   │
│  response_model_exclude_unset   │  .model_dump(exclude_unset=..)│
│    (already shown in § 6.4)     │                               │
│                                 │                               │
│  These route-level params are   │  Pydantic internally calls    │
│  FastAPI's shortcut.            │  model_dump() with them.      │
│                                 │                               │
│  WHEN TO USE WHICH:             │                               │
│  Route params → different       │  model_dump() in code → when  │
│  endpoints need different        │  you need the dict in your    │
│  shapes of the same model       │  Python logic, not just the   │
│                                 │  HTTP response                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical comparison — two endpoints, same model, different shapes:**

```python
class UserRead(BaseModel):
    id: int
    username: str
    email: str
    role: str
    internal_notes: str     # Admin-only field

# Public endpoint — exclude internal_notes
@app.get("/users/{id}", response_model=UserRead,
         response_model_exclude={"internal_notes"})
async def get_user_public(user_id: int):
    return users[user_id]

# Admin endpoint — include everything
@app.get("/admin/users/{id}", response_model=UserRead)
async def get_user_admin(user_id: int):
    return users[user_id]
```

> "This is cleaner than creating two separate response models (`UserPublic` and `UserAdmin`) when the only difference is which fields to expose. However, if the schemas diverge significantly, separate models remain the better design."

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 6.5                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  class ArticleRead(BaseModel):                                  │
│      model_config = ConfigDict(populate_by_name=True)           │
│      id: int                                                    │
│      title: str                                                 │
│      body: str                                                  │
│      author_id: int                                             │
│      published_at: str = Field(alias="publishedAt")             │
│      internal_draft_notes: str                                  │
│                                                                 │
│  Write two GET endpoints:                                       │
│  1. /articles/{id} — public: exclude internal_draft_notes,      │
│     use camelCase aliases in response                           │
│  2. /admin/articles/{id} — full data, no aliases                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

from fastapi import FastAPI
from pydantic import BaseModel, ConfigDict, Field

app = FastAPI()

class ArticleRead(BaseModel):
    model_config = ConfigDict(populate_by_name=True)
    id: int
    title: str
    body: str
    author_id: int
    published_at: str = Field(alias="publishedAt")
    internal_draft_notes: str

articles: dict[int, dict] = {
    1: {
        "id": 1, "title": "Hello", "body": "Content", "author_id": 5,
        "published_at": "2025-01-01", "internal_draft_notes": "needs review"
    }
}

# 1. Public endpoint — camelCase aliases, no internal notes
@app.get(
    "/articles/{article_id}",
    response_model=ArticleRead,
    response_model_exclude={"internal_draft_notes"},
    response_model_by_alias=True,
)
async def get_article_public(article_id: int):
    return articles[article_id]
    # Response: {"id": 1, "title": "Hello", ..., "publishedAt": "2025-01-01"}
    #           ← camelCase alias used, internal_draft_notes excluded

# 2. Admin endpoint — all fields, snake_case field names
@app.get("/admin/articles/{article_id}", response_model=ArticleRead)
async def get_article_admin(article_id: int):
    return articles[article_id]
    # Response: {"id": 1, "title": "Hello", ...,
    #            "published_at": "2025-01-01",
    #            "internal_draft_notes": "needs review"}
```

---

## 6.6 Generic Models — Response[T] Pattern

**The problem: every API endpoint returns a different payload. Without generics, you create a wrapper model for each:**

```python
# ❌ Without generics — repeated boilerplate
class UserListResponse(BaseModel):
    items: list[UserRead]
    total: int
    page: int

class ProductListResponse(BaseModel):
    items: list[ProductRead]
    total: int
    page: int

# Same structure. Different type. Duplicated every time you add an entity.
```

**Generic models let you write the shape once and parameterise the type:**

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    per_page: int
```

**Using it — T becomes the concrete type you provide:**

```python
# PaginatedResponse[UserRead] — T is replaced with UserRead
user_page: PaginatedResponse[UserRead] = PaginatedResponse(
    items=[UserRead(id=1, name="Alice")],
    total=50,
    page=1,
    per_page=20,
)

# PaginatedResponse[ProductRead] — T is replaced with ProductRead
product_page: PaginatedResponse[ProductRead] = PaginatedResponse(
    items=[ProductRead(id=1, name="Widget", price=9.99)],
    total=120,
    page=1,
    per_page=20,
)

# Both are validated — T is enforced
print(type(user_page.items[0]))      # <class 'UserRead'>
print(type(product_page.items[0]))   # <class 'ProductRead'>
```

**A standard API response wrapper — another common pattern:**

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")

class APIResponse(BaseModel, Generic[T]):
    success: bool
    data: T | None = None
    message: str = ""
    errors: list[str] = []
```

```python
# Success case
ok_response: APIResponse[UserRead] = APIResponse(
    success=True,
    data=UserRead(id=1, name="Alice"),
)

# Error case — data is None
err_response: APIResponse[UserRead] = APIResponse(
    success=False,
    errors=["User not found"],
)

print(ok_response.model_dump())
# {"success": True, "data": {"id": 1, "name": "Alice"},
#  "message": "", "errors": []}

print(err_response.model_dump())
# {"success": False, "data": None, "message": "", "errors": ["User not found"]}
```

**Using Generic models in FastAPI:**

```python
@app.get("/users", response_model=PaginatedResponse[UserRead])
async def list_users(page: int = 1, per_page: int = 20):
    # ...fetch from DB...
    return PaginatedResponse(
        items=[...],
        total=total_count,
        page=page,
        per_page=per_page,
    )
```

> "FastAPI and Swagger UI understand generic types. `PaginatedResponse[UserRead]` generates a schema where `items` is specifically typed as a list of `UserRead` objects — with correct field names and types shown in the docs."

**Validation is fully T-aware:**

```python
# items must be a list of UserRead objects (or dicts that coerce to UserRead)
PaginatedResponse[UserRead](
    items=[{"id": 1, "name": "Alice"}, {"id": "not-int", "name": "Bob"}],
    total=2, page=1, per_page=10
)
# ValidationError: items.1.id
#   Input should be a valid integer
```

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 6.6                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Using the APIResponse[T] pattern from this section:            │
│                                                                 │
│  class ProductRead(BaseModel):                                  │
│      id: int                                                    │
│      name: str                                                  │
│      price: float                                               │
│                                                                 │
│  1. Write a FastAPI endpoint GET /products/{id} that returns    │
│     APIResponse[ProductRead] with success=True and the product. │
│                                                                 │
│  2. What does the Swagger schema look like for the "data"       │
│     field? Is it typed as ProductRead or just "any"?            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

from typing import Generic, TypeVar
from pydantic import BaseModel
from fastapi import FastAPI, HTTPException

T = TypeVar("T")

class APIResponse(BaseModel, Generic[T]):
    success: bool
    data: T | None = None
    message: str = ""
    errors: list[str] = []

class ProductRead(BaseModel):
    id: int
    name: str
    price: float

app = FastAPI()

_products = {1: ProductRead(id=1, name="Widget", price=9.99)}

# 1. Endpoint returning APIResponse[ProductRead]
@app.get("/products/{product_id}", response_model=APIResponse[ProductRead])
async def get_product(product_id: int):
    if product_id not in _products:
        return APIResponse(success=False, errors=["Product not found"])
    return APIResponse(success=True, data=_products[product_id])

# 2. Swagger schema: the "data" field is typed as ProductRead schema,
#    not as "any". Swagger will show it with the correct nested schema:
#    {
#      "data": {
#        "$ref": "#/components/schemas/ProductRead"
#      }
#    }
#    Generic types are fully resolved by FastAPI for OpenAPI generation.
```

---

## 6.7 Discriminated Unions

**The problem: your API can receive different object shapes depending on context — a payment can be a credit card, PayPal, or bank transfer. How does Pydantic know which model to use?**

```python
from typing import Union
from pydantic import BaseModel

class CreditCard(BaseModel):
    card_number: str
    expiry: str

class PayPal(BaseModel):
    email: str

# ❌ Without a discriminator — Pydantic must try every model
Payment = Union[CreditCard, PayPal]

class Checkout(BaseModel):
    amount: float
    payment: Payment
```

**The problem without a discriminator — order-dependent and ambiguous:**

```python
# Pydantic tries CreditCard first, then PayPal
checkout = Checkout(amount=99.99, payment={"email": "alice@example.com"})
# ValidationError — CreditCard tried first, fails on missing card_number
# Pydantic then tries PayPal... but the error message is confusing
```

**The fix: add a `Literal` discriminator field and tell Pydantic which field to look at:**

```python
from typing import Literal, Union
from pydantic import BaseModel, Field

class CreditCardPayment(BaseModel):
    method: Literal["credit_card"]   # ← Literal type: exactly this string
    card_number: str
    expiry: str
    cvv: str

class PayPalPayment(BaseModel):
    method: Literal["paypal"]
    email: str

class BankTransferPayment(BaseModel):
    method: Literal["bank_transfer"]
    account_number: str
    routing_number: str

class Checkout(BaseModel):
    amount: float
    payment: Union[
        CreditCardPayment,
        PayPalPayment,
        BankTransferPayment,
    ] = Field(discriminator="method")  # ← look at the "method" field first
```

**Pydantic reads the discriminator field and jumps directly to the right model:**

```python
# ✅ Pydantic sees method="paypal" → validates as PayPalPayment immediately
checkout = Checkout(
    amount=49.99,
    payment={"method": "paypal", "email": "alice@example.com"}
)
print(type(checkout.payment))          # <class 'PayPalPayment'>
print(checkout.payment.email)          # alice@example.com

# ✅ Different shape, same field
checkout2 = Checkout(
    amount=99.99,
    payment={
        "method": "credit_card",
        "card_number": "4111-1111-1111-1111",
        "expiry": "12/28",
        "cvv": "123"
    }
)
print(type(checkout2.payment))         # <class 'CreditCardPayment'>

# ❌ Unknown method value → clear error
Checkout(amount=10, payment={"method": "bitcoin", "address": "1A2B..."})
# ValidationError: payment → method — Input should be 'credit_card',
#   'paypal' or 'bank_transfer'
```

**Why discriminated unions are better than plain Union:**

```
┌─────────────────────────────────────────────────────────────────┐
│         Union  vs  Discriminated Union                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Plain Union[A, B, C]:                                          │
│  ─────────────────────                                          │
│  Pydantic tries A first. If it fails, tries B. Then C.          │
│  SLOW — O(n) validation attempts.                               │
│  CONFUSING errors — the error from A, not from the right model. │
│  FRAGILE — field overlap between models can mismatch silently.  │
│                                                                 │
│  Discriminated Union (with Literal field):                      │
│  ──────────────────────────────────────────                     │
│  Pydantic reads the discriminator field. Jumps to the right     │
│  model directly.                                                │
│  FAST — O(1) model selection.                                   │
│  CLEAR errors — error comes from the correct model.             │
│  SAFE — unknown discriminator values fail immediately.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Alternative syntax — using Annotated:**

```python
from typing import Annotated, Literal, Union
from pydantic import Discriminator

# This is equivalent to Field(discriminator="method") above
Payment = Annotated[
    Union[CreditCardPayment, PayPalPayment, BankTransferPayment],
    Discriminator("method")
]

class Checkout(BaseModel):
    amount: float
    payment: Payment    # Use the type alias directly
```

> "Use the `Field(discriminator=...)` form when the discriminated union is one field in a larger model. Use the `Annotated[..., Discriminator(...)]` form when you want to define a reusable polymorphic type alias that you can use in multiple places."

---

```
┌─────────────────────────────────────────────────────────────────┐
│               ✎ PRACTICE CHECKPOINT — 6.7                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Build a notification system:                                   │
│                                                                 │
│  - EmailNotification: type="email", recipient_email: str,       │
│    subject: str                                                 │
│  - SmsNotification: type="sms", phone_number: str,              │
│    message: str                                                 │
│  - PushNotification: type="push", device_token: str,            │
│    title: str, body: str                                        │
│                                                                 │
│  Build a NotificationRequest model with:                        │
│  - user_id: int                                                 │
│  - notification: Union[...] = Field(discriminator="type")       │
│                                                                 │
│  Test that passing type="fax" gives a clear validation error.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ▼ SOLUTION

from typing import Literal, Union
from pydantic import BaseModel, Field

class EmailNotification(BaseModel):
    type: Literal["email"]
    recipient_email: str
    subject: str

class SmsNotification(BaseModel):
    type: Literal["sms"]
    phone_number: str
    message: str

class PushNotification(BaseModel):
    type: Literal["push"]
    device_token: str
    title: str
    body: str

class NotificationRequest(BaseModel):
    user_id: int
    notification: Union[
        EmailNotification,
        SmsNotification,
        PushNotification,
    ] = Field(discriminator="type")

# ✅ Correct usage
req = NotificationRequest(
    user_id=42,
    notification={"type": "sms", "phone_number": "+1555123456", "message": "Hello!"}
)
print(type(req.notification))          # <class 'SmsNotification'>
print(req.notification.phone_number)   # +1555123456

# ❌ Unknown type — clear, immediate error
try:
    NotificationRequest(
        user_id=1,
        notification={"type": "fax", "fax_number": "555-1234"}
    )
except Exception as e:
    print(e)
# ValidationError: notification → type
#   Input should be 'email', 'sms' or 'push'
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│           PYDANTIC QUICK REFERENCE — NEW IN THIS REVISION       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BUILT-IN TYPES (pydantic[email] for EmailStr):                 │
│      email: EmailStr        # validated email                   │
│      url: HttpUrl           # http/https URL (Url object)       │
│      count: PositiveInt     # int > 0                           │
│      amount: PositiveFloat  # float > 0                         │
│      stock: NonNegativeInt  # int >= 0                          │
│                                                                 │
│  STRICT TYPES (per-field, no model config needed):              │
│      from pydantic import StrictInt, StrictStr, StrictFloat     │
│      age: StrictInt         # no "25" → 25 coercion             │
│                                                                 │
│  ENUMS:                                                         │
│      class Status(str, Enum):  # Always use (str, Enum)         │
│          active = "active"                                      │
│          inactive = "inactive"                                  │
│                                                                 │
│  TYPEADAPTER:                                                   │
│      adapter = TypeAdapter(list[int])  # define at module level │
│      adapter.validate_python(["1","2"]) # → [1, 2]              │
│      adapter.validate_json("[1,2]")                             │
│                                                                 │
│  RECURSIVE MODELS:                                              │
│      class Node(BaseModel):                                     │
│          children: list["Node"] = []  # string forward ref      │
│      Node.model_rebuild()             # required after class    │
│                                                                 │
│  ANNOTATED VALIDATORS:                                          │
│      MyType = Annotated[str,                                    │
│          BeforeValidator(transform_fn),                         │
│          Field(max_length=50),                                  │
│          AfterValidator(validate_fn)]                           │
│                                                                 │
│  COMPUTED FIELDS:                                               │
│      @computed_field                  # outer decorator         │
│      @property                        # inner decorator         │
│      def full_name(self) -> str:  ... # return type REQUIRED    │
│                                                                 │
│  FIELD SERIALIZER:                                              │
│      @field_serializer("field_name")                            │
│      def serialize_field(self, v: FieldType) -> SerType: ...    │
│                                                                 │
│  model_post_init:                                               │
│      def model_post_init(self, __context) -> None:              │
│          self.computed_field = expensive_fn(self.input)         │
│                                                                 │
│  PRIVATE ATTRIBUTES:                                            │
│      _cache: dict = PrivateAttr(default_factory=dict)           │
│      # Not in model_dump(), not in __init__ signature           │
│                                                                 │
│  model_copy:                                                    │
│      copy = model.model_copy(update={"field": value})           │
│      copy = model.model_copy(deep=True)  # deep copy            │
│      # WARNING: update values are NOT validated                 │
│                                                                 │
│  GENERIC MODELS:                                                │
│      T = TypeVar("T")                                           │
│      class Page(BaseModel, Generic[T]):                         │
│          items: list[T]                                         │
│          total: int                                             │
│      # Usage: Page[UserRead], Page[ProductRead]                 │
│                                                                 │
│  DISCRIMINATED UNIONS:                                          │
│      payment: Union[CardPay, PayPal] = Field(                   │
│          discriminator="method")      # Literal field required  │
│                                                                 │
│  COMMON MISTAKES (additions):                                   │
│      ❌ model_copy(update=...) skips validation                 │
│      ❌ Forgot @property under @computed_field                  │
│      ❌ No return type on @computed_field → PydanticUserError   │
│      ❌ Forgot model_rebuild() on recursive model               │
│      ❌ PrivateAttr name doesn't start with _ → ignored         │
│      ❌ EmailStr without pydantic[email] installed               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  PYDANTIC = YOUR API'S GATEKEEPER                               │
│                                                                 │
│  Every piece of data that enters or leaves your API             │
│  passes through Pydantic models.                                │
│                                                                 │
│                                                                 │
│   Client JSON ──▶ [GATEKEEPER IN] ──▶ Your Code                │
│                    Validates                                    │
│                    Coerces types                                │
│                    Rejects bad data                             │
│                                                                 │
│   Your Code ──▶ [GATEKEEPER OUT] ──▶ Client JSON               │
│                  Filters fields                                 │
│                  Applies aliases                                │
│                  Strips secrets                                 │
│                                                                 │
│                                                                 │
│  THE THREE PILLARS:                                             │
│  ├─ VALIDATION:   Data is checked against rules on arrival      │
│  ├─ COERCION:     Sensible type conversions happen auto         │
│  └─ SERIALIZATION: Data is shaped on departure                  │
│                                                                 │
│  THE THREE MODEL PATTERN:                                       │
│  ├─ Create model: What the client sends (no id, no timestamps) │
│  ├─ Read model:   What the server returns (id, timestamps,     │
│  │                no secrets)                                   │
│  └─ Update model: What the client can change (all optional)    │
│                                                                 │
│  FROM WEEK 1 TO NOW:                                            │
│  ├─ Type hints (Week 1)     → They ARE the Pydantic schema     │
│  ├─ Dataclasses (Week 1)    → BaseModel is their evolution     │
│  ├─ Decorators (Week 1)     → @field_validator, @model_valid.. │
│  └─ FastAPI (This week)     → Pydantic IS the I/O layer        │
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
│  THIS WEEK (Lecture 4 — Error Handling & Dependencies):         │
│  └─ Custom exception handlers for ValidationError               │
│     Turn Pydantic errors into proper HTTP error responses       │
│                                                                 │
│  THIS WEEK (Lecture 5 — Testing):                               │
│  └─ Test that validation rejects bad data                       │
│     Test that valid data creates correct objects                 │
│                                                                 │
│  WEEK 2 PROJECT (Task Manager API):                             │
│  └─ You'll use everything from this lecture:                    │
│     Create/Read/Update models for Tasks and Categories          │
│     Field constraints for priority, status, due dates           │
│     Validators for business rules                               │
│                                                                 │
│  WEEK 4 (External APIs):                                        │
│  └─ Pydantic validates external API responses                   │
│     Never trust data from outside — same gatekeeper principle   │
│                                                                 │
│  WEEK 5 (Authentication):                                       │
│  └─ User registration with password validation                  │
│     Token payload as Pydantic model                             │
│                                                                 │
│  WEEK 12 (Production):                                          │
│  └─ pydantic-settings for configuration management              │
│     Environment variables validated by Pydantic                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```