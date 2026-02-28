# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BREAK IT FIRST, FIX IT SECOND                                  │
│  ─────────────────────────────                                  │
│  Students will watch a "working" integration CRASH when an      │
│  external API changes shape. Feel the fragility. Then learn     │
│  the architecture that prevents it.                             │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Your system is a warehouse. External APIs are foreign          │
│  suppliers. The transformation layer is the receiving dock.     │
│  Every concept maps to this supply chain.                       │
│                                                                 │
│  DEFENSE IN DEPTH                                               │
│  ───────────────                                                │
│  This lecture teaches healthy architectural paranoia.            │
│  Never trust data you didn't create.                            │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Pydantic (Week 3) → now used DEFENSIVELY, not descriptively   │
│  Repository pattern (Week 6) → where sync logic lives          │
│  httpx (Lecture 1) → was about the transport; now about cargo  │
│  Circuit breaker (Lecture 2) → handled network failure;         │
│      this lecture handles DATA failure                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│               DATA TRANSFORMATION & INTEGRATION                 │
│                     (3–4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (40 min)                                   │
│  ├─ 1.1 The Fragile Integration (Demonstration)                 │
│  ├─ 1.2 The Trust Boundary                                      │
│  ├─ 1.3 External vs Internal: Two Separate Worlds               │
│  └─ 1.4 The Warehouse Receiving Dock Analogy                    │
│                                                                 │
│  PART 2: DEFENSIVE PARSING (50 min)                             │
│  ├─ 2.1 Pydantic as Your Inspection Team                        │
│  ├─ 2.2 Handling Missing Fields                                 │
│  ├─ 2.3 Handling Extra and Unexpected Fields                    │
│  ├─ 2.4 Nested External Responses                               │
│  └─ 2.5 Validators as Guardrails                                │
│                                                                 │
│  PART 3: DATA NORMALIZATION (50 min)                            │
│  ├─ 3.1 N Suppliers, One Warehouse Format                       │
│  ├─ 3.2 The Internal Model                                      │
│  ├─ 3.3 The Adapter Pattern                                     │
│  ├─ 3.4 The Complete Transformation Pipeline                    │
│  └─ 3.5 Error Recovery in the Pipeline                          │
│                                                                 │
│  PART 4: PERSISTENCE (40 min)                                   │
│  ├─ 4.1 Why Store External Data Locally?                        │
│  ├─ 4.2 Designing the Storage Model                             │
│  ├─ 4.3 Upsert: Create or Update in One Query                  │
│  └─ 4.4 Sync Strategies                                         │
│                                                                 │
│  PART 5: BACKGROUND REFRESH (30 min)                            │
│  ├─ 5.1 The Stale Data Problem                                  │
│  ├─ 5.2 FastAPI BackgroundTasks                                 │
│  ├─ 5.3 Refresh Patterns                                        │
│  └─ 5.4 When BackgroundTasks Isn't Enough                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Fragile Integration

**Start with a demonstration. Show them code that works today and dies tomorrow.**

```python
# demo_fragile.py — Run this with students watching

# This is what httpx gives you after resp.json() — a plain dict.
# Simulating the OpenWeatherMap API response:
api_response = {
    "name": "London",
    "main": {"temp": 293.15, "humidity": 65},
    "weather": [{"id": 500, "description": "light rain"}],
    "wind": {"speed": 5.2},
}

def extract_weather(data: dict) -> dict:
    """Quick and dirty extraction. Looks fine, right?"""
    return {
        "city": data["name"],
        "temp_celsius": round(data["main"]["temp"] - 273.15, 1),
        "description": data["weather"][0]["description"],
        "humidity": data["main"]["humidity"],
        "wind_speed": data["wind"]["speed"],
    }

result = extract_weather(api_response)
print(result)
# {'city': 'London', 'temp_celsius': 20.0, 'description': 'light rain',
#  'humidity': 65, 'wind_speed': 5.2}
```

**Works. Ship it. Move on to the next feature.**

**Three months later, the API provider "improves" their API...**

```python
# Scenario 1: A field gets restructured
# Provider merged "wind" into "main" in their v2 update

api_response_v2 = {
    "name": "London",
    "main": {"temp": 293.15, "humidity": 65, "wind_speed": 5.2},
    "weather": [{"id": 500, "description": "light rain"}],
    # "wind" key is GONE
}

extract_weather(api_response_v2)
# 💥 KeyError: 'wind'
# Your production server returns 500. Users see nothing.
```

```python
# Scenario 2: A field becomes nullable for some inputs
# Remote weather stations don't always report all fields

api_response_v3 = {
    "name": "McMurdo Station",
    "main": {"temp": 245.0, "humidity": None},
    "weather": [],
    "wind": {"speed": 12.4},
}

extract_weather(api_response_v3)
# 💥 IndexError: list index out of range
# data["weather"][0] — the list is empty for this location
# And if you fixed that: humidity is None. Store it, do math on it later → TypeError
```

**Now the truly dangerous one:**

```python
# Scenario 3: The SILENT killer — units changed, no error raised

api_response_v4 = {
    "name": "London",
    "main": {"temp": 20.0, "humidity": 65},   # Was Kelvin. Now Celsius.
    "weather": [{"id": 500, "description": "light rain"}],
    "wind": {"speed": 5.2},
}

result = extract_weather(api_response_v4)
print(result)
# {'city': 'London', 'temp_celsius': -253.1, 'description': 'light rain', ...}
#
# No error. No crash. Just WRONG DATA silently stored in your database.
# London is not -253°C. But your system doesn't know that.  🧊
```

**Now ask the class:**

> "How many of you would catch that last one before production? No KeyError. No TypeError. No stack trace. Just wrong numbers served to your users, stored in your database, used for decisions. This is the scariest class of bug — silent data corruption."

**The insight:**

> "Your httpx client knows how to *send requests* reliably (Lecture 1). Your circuit breaker knows *when to stop trying* (Lecture 2). But neither of them can tell you whether the DATA that comes back makes any sense. Lectures 1 and 2 secured the transport. This lecture secures the cargo."

---

## 1.2 The Trust Boundary

**There is a line in your architecture. On one side: chaos. On the other: your system.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      THE TRUST BOUNDARY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│     OUTSIDE YOUR CONTROL              YOUR SYSTEM               │
│     ─────────────────────             ──────────                │
│                                                                 │
│  ┌──────────────┐              ┌──────────────────────┐         │
│  │ Provider A   │              │                      │         │
│  │ (their API,  │── JSON ────▶ │    TRUST BOUNDARY    │         │
│  │  their rules)│              │                      │         │
│  └──────────────┘              │  ┌────────────────┐  │         │
│  ┌──────────────┐              │  │  Validate      │  │         │
│  │ Provider B   │── JSON ────▶ │  │  Transform     │──▶ Clean   │
│  │ (different   │              │  │  Normalize     │  │ internal│
│  │  format!)    │              │  └────────────────┘  │ models  │
│  └──────────────┘              │                      │         │
│  ┌──────────────┐              │  Reject garbage ──▶ ❌│         │
│  │ Provider C   │── JSON ────▶ │                      │         │
│  │ (changes     │              └──────────────────────┘         │
│  │  without     │                                               │
│  │  warning)    │                                               │
│  └──────────────┘                                               │
│                                                                 │
│  You control NOTHING            You control EVERYTHING          │
│  on this side.                  on this side.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What crosses the trust boundary is raw JSON — a `dict` from `resp.json()`. It has:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  NO guarantees about:                                           │
│  ├─ Which fields are present                                    │
│  ├─ What types the values are                                   │
│  ├─ Whether nested structures are consistent                    │
│  ├─ What units the numbers use                                  │
│  ├─ Whether the format has changed since last week              │
│  └─ Whether the documentation is even accurate                  │
│                                                                 │
│  Your job at the trust boundary:                                │
│  ├─ Parse the incoming data into a KNOWN structure              │
│  ├─ Reject anything that doesn't make sense                     │
│  ├─ Convert to YOUR internal representation                     │
│  └─ Never let raw external data leak into your business logic   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 External vs Internal: Two Separate Worlds

**This is the core architectural insight of this lecture.**

Your system now has TWO distinct categories of data model. You used Pydantic models in Week 3 for *your own* API's request and response schemas. Those were about defining *your* contract with *your* clients. Now you need models that represent *someone else's* contract with *you*.

```
┌─────────────────────────────────────────────────────────────────┐
│              EXTERNAL MODELS VS INTERNAL MODELS                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EXTERNAL MODELS                   INTERNAL MODELS              │
│  ───────────────                   ───────────────              │
│  Shape: matches THEIR API          Shape: matches YOUR domain   │
│  Purpose: parse & validate         Purpose: business logic      │
│  Lifetime: trust boundary only     Lifetime: your entire app    │
│  Changes: when THEY change         Changes: when YOU decide     │
│  Naming: mirrors their field       Naming: your conventions     │
│          names exactly                                          │
│                                                                 │
│                                                                 │
│  Provider A response:              ┌──────────────────┐         │
│  {                                 │  WeatherData     │         │
│    "main": {"temp": 293.15}  ────▶ │  ├─ city: str    │         │
│    "weather": [...]                │  ├─ temp_c: float │         │
│  }                                 │  ├─ desc: str     │         │
│                                    │  └─ wind_ms: float│         │
│  Provider B response:              │                   │         │
│  {                                 │  ONE model for    │         │
│    "current": {              ────▶ │  ALL providers    │         │
│      "temp_c": 20.0               │                   │         │
│    }                               └──────────────────┘         │
│  }                                                              │
│                                                                 │
│  Different shapes ─── TRANSFORMATION ───▶ Same shape            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The immediate objection students will have:**

> "Isn't this code duplication? I'm writing models that look almost the same twice."

No. They look the same today by coincidence. External models are shaped by someone else's decisions and can change without warning. Internal models are shaped by *your* domain and change only when *you* decide. Coupling them together means that a provider renaming a field forces changes across your business logic, database queries, tests, and API responses. Separation means a provider change touches *exactly one file* — the external model and its adapter.

---

## 1.4 The Warehouse Receiving Dock Analogy

**This analogy carries through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE WAREHOUSE RECEIVING DOCK                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your APPLICATION is a WAREHOUSE.                               │
│  External APIs are INTERNATIONAL SUPPLIERS.                     │
│                                                                 │
│  Supplier A ships in wooden crates with labels in Japanese.     │
│  Supplier B ships in cardboard boxes with labels in German.     │
│  Supplier C ships in plastic wrap with no labels at all.        │
│                                                                 │
│  The RECEIVING DOCK:                                            │
│  1. Inspects each shipment       (validation)                   │
│  2. Reads the foreign labels     (external model parsing)       │
│  3. Repackages into YOUR boxes   (normalization)                │
│  4. Applies YOUR barcode labels  (internal model)               │
│  5. Stores on YOUR shelves       (database)                     │
│  6. Rejects damaged goods        (error handling)               │
│  7. Reorders when stock is low   (background refresh)           │
│                                                                 │
│  You NEVER store foreign-labeled boxes on your shelves.         │
│  You NEVER make warehouse staff learn Japanese just to          │
│  read one supplier's labels.                                    │
│  You have ONE labeling system. Every supplier adapts to it.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to programming:**

```
Warehouse                   │  Your Application
────────────────────────────│───────────────────────────────
Warehouse                   │  Your system
International suppliers     │  External APIs
Shipping labels / packaging │  API response format (JSON shape)
Receiving dock              │  Transformation layer
Inspection checklist        │  External Pydantic model
Your barcode labels         │  Internal domain model
Repackaging workers         │  Adapter classes
Shelves                     │  Database (PostgreSQL)
Damaged/missing goods       │  Invalid or malformed data
Inventory count update      │  Upsert operation
Sell-by date on goods       │  Data freshness (fetched_at)
Reordering when stock low   │  Background data refresh
```

---

# PART 2: DEFENSIVE PARSING

## 2.1 Pydantic as Your Inspection Team

**Mindset shift from Week 3:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TWO MINDSETS FOR PYDANTIC                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 3 MINDSET: PRESCRIPTIVE                                   │
│  ─────────────────────────────                                  │
│  "I define the rules. I decide the shape.                       │
│   If the client sends bad data, that's a 422."                  │
│                                                                 │
│     class CreateTask(BaseModel):                                │
│         title: str                                              │
│         priority: int = Field(ge=1, le=5)                       │
│                                                                 │
│  You are the AUTHORITY. Your model is the LAW.                  │
│                                                                 │
│                                                                 │
│  WEEK 8 MINDSET: DEFENSIVE                                      │
│  ─────────────────────────                                      │
│  "I describe what I EXPECT to receive.                          │
│   If reality differs, I handle it gracefully."                  │
│                                                                 │
│     class OpenWeatherResponse(BaseModel):                       │
│         model_config = ConfigDict(extra="ignore")               │
│         name: str                                               │
│         wind: OpenWeatherWind | None = None                     │
│                                                                 │
│  You are the INSPECTOR. Their data is the unknown.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In Week 3, FastAPI parsed request bodies into Pydantic models automatically. Now you parse manually:**

```python
import httpx
from pydantic import BaseModel, ConfigDict, ValidationError

# Step 1: Make the HTTP request (Lecture 1 skills)
async with httpx.AsyncClient() as client:
    resp = await client.get("https://api.openweathermap.org/data/2.5/weather", ...)
    raw_data: dict = resp.json()  # This is an untyped dict. No safety.

# Step 2: Parse into a typed, validated external model
try:
    external = OpenWeatherResponse.model_validate(raw_data)
    # Now you have typed, validated data. Safe to access attributes.
except ValidationError as e:
    # The external API sent data that doesn't match your expectations.
    # This is NOT a client error — it's an integration problem.
    logger.error(f"Provider schema mismatch: {e}")
```

**`model_validate()` is the entry point for external data.** It takes a raw dict and returns a typed model instance — or raises `ValidationError` if the data doesn't match. This is your inspection checkpoint.

**Why not just use `.get()` with defaults?**

```python
# ❌ Tempting but dangerous:
city = data.get("name", "Unknown")
temp = data.get("main", {}).get("temp", 0)

# Problems:
# 1. No type checking — temp could be a string and you'd never know
# 2. Silent defaults mask real problems — is "Unknown" a valid city?
# 3. Is 0 a valid default temperature? 0°K means something very different from "missing"
# 4. No validation — temp could be -9999 (a common "no data" sentinel)
# 5. Nested .get() chains are unreadable at depth 3+

# ✅ Pydantic: validates types, rejects garbage, documents the expected shape
external = OpenWeatherResponse.model_validate(data)
# If "name" is missing → ValidationError (you KNOW about it)
# If "temp" is a string → ValidationError (you KNOW about it)
# If "temp" is -9999    → your validator catches it (Part 2.5)
```

---

## 2.2 Handling Missing Fields

**External APIs remove fields, add optional ones, and return `null` where they used to return values. Your models must survive this.**

```python
from pydantic import BaseModel, ConfigDict

class OpenWeatherWind(BaseModel):
    speed: float
    deg: int | None = None      # Wind direction — not always present
    gust: float | None = None   # Gust speed — only during storms

class OpenWeatherMain(BaseModel):
    temp: float                           # Always present (we hope)
    humidity: int | None = None           # Sometimes null for remote stations
    pressure: float | None = None         # Not all stations report this
    feels_like: float | None = None       # Added in API v2.5, older responses lack it
    temp_min: float | None = None         # Optional in some response modes
    temp_max: float | None = None
```

**The pattern: any field that MIGHT be absent or null gets `| None = None`.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 FIELD OPTIONALITY STRATEGY                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  KEEP REQUIRED only if you are CERTAIN the field is always      │
│  present AND your system CANNOT function without it.            │
│                                                                 │
│  Use  | None = None  for everything else. Why?                  │
│                                                                 │
│  Scenario: Provider always returns "humidity"... until they     │
│  don't. If humidity is required in your external model:         │
│                                                                 │
│    Required field → ValidationError → data lost entirely        │
│    (you reject the whole response because of one missing field) │
│                                                                 │
│  If humidity is optional:                                       │
│                                                                 │
│    Optional field → None → rest of data still usable            │
│    (you get the temperature, description, wind — just not       │
│     humidity. Partial data is better than no data.)             │
│                                                                 │
│  Rule of thumb:                                                 │
│  ├─ Required: fields you CANNOT build an internal model without │
│  │            (e.g., the city name, the temperature)            │
│  └─ Optional: everything else — be generous with optionality   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Handling Extra and Unexpected Fields

**APIs add fields constantly. Your models shouldn't break when they do.**

```python
# Suppose you built your model based on today's API docs:
class OpenWeatherResponse(BaseModel):
    name: str
    main: OpenWeatherMain
    weather: list[OpenWeatherCondition]
    wind: OpenWeatherWind | None = None

# Next month, the API adds: "air_quality", "alerts", "forecast_link"
# If your model has the default Pydantic behavior:
#   → ValidationError on extra fields? No, defaults allow extras.
#   → But being EXPLICIT is better.
```

**Be explicit about your `extra` strategy:**

```python
class OpenWeatherResponse(BaseModel):
    model_config = ConfigDict(extra="ignore")  # ← THE KEY SETTING
    
    name: str
    main: OpenWeatherMain
    weather: list[OpenWeatherCondition]
    wind: OpenWeatherWind | None = None
```

**Three options for `extra` — and when to use each:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     EXTRA FIELD STRATEGIES                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  extra="ignore"  (DEFAULT CHOICE FOR EXTERNAL DATA)             │
│  ──────────────                                                 │
│  Unknown fields silently dropped. Model only keeps what you     │
│  defined. Safe and predictable.                                 │
│  Use for: External API responses (almost always this one)       │
│                                                                 │
│  extra="allow"                                                  │
│  ─────────────                                                  │
│  Unknown fields kept as extra data. Accessible via              │
│  model.model_extra. Useful if you want to LOG what the          │
│  API is sending without parsing every field.                    │
│  Use for: Debugging or when you need to forward raw fields      │
│                                                                 │
│  extra="forbid"  (WEEK 3 DEFAULT — YOUR OWN API)               │
│  ──────────────                                                 │
│  Unknown fields raise ValidationError. Strict contract.         │
│  Use for: YOUR API's request models (Week 3)                    │
│  NEVER use for: External API responses (they add fields)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "In Week 3, `extra='forbid'` was your friend — it caught clients sending garbage to YOUR API. For external data, `extra='forbid'` is your enemy — it crashes your system every time a provider adds a field you haven't seen yet."

---

## 2.4 Nested External Responses

**Real API responses are rarely flat. Model the full nesting.**

Here is a second provider — WeatherAPI.com — with a completely different structure for the same type of data. Notice how different the shape is from OpenWeatherMap:

```python
# External models for WeatherAPI.com (Provider B)
# These mirror THEIR response shape exactly.

class WeatherAPICondition(BaseModel):
    model_config = ConfigDict(extra="ignore")
    
    text: str
    icon: str | None = None
    code: int | None = None

class WeatherAPICurrent(BaseModel):
    model_config = ConfigDict(extra="ignore")
    
    temp_c: float
    temp_f: float | None = None
    humidity: int
    condition: WeatherAPICondition
    wind_kph: float
    wind_dir: str | None = None
    feelslike_c: float | None = None
    uv: float | None = None

class WeatherAPILocation(BaseModel):
    model_config = ConfigDict(extra="ignore")
    
    name: str
    country: str
    lat: float
    lon: float
    localtime: str | None = None

class WeatherAPIResponse(BaseModel):
    model_config = ConfigDict(extra="ignore")
    
    location: WeatherAPILocation
    current: WeatherAPICurrent
```

**Notice the difference from OpenWeatherMap:**

```
┌─────────────────────────────────────────────────────────────────┐
│               SAME DATA, DIFFERENT SHAPES                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OpenWeatherMap:                 WeatherAPI.com:                │
│  ──────────────                  ───────────────                │
│  {                               {                              │
│    "name": "London",               "location": {               │
│    "main": {                         "name": "London",          │
│      "temp": 293.15   ← Kelvin       "country": "UK"           │
│    },                              },                           │
│    "weather": [        ← list      "current": {                │
│      {"description":                 "temp_c": 20.0  ← Celsius │
│        "light rain"}                 "condition": {  ← object  │
│    ],                                  "text":                  │
│    "wind": {                             "Light rain"           │
│      "speed": 5.2     ← m/s         },                         │
│    }                                 "wind_kph": 18.7 ← km/h   │
│  }                                 }                            │
│                                  }                              │
│                                                                 │
│  Temperature in Kelvin          Temperature in Celsius          │
│  Weather as a list              Condition as an object          │
│  Wind in meters/second          Wind in kilometers/hour         │
│  Flat-ish structure             Deeply nested structure         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Same concept — current weather for a city. Completely different JSON shapes. Different units. Different nesting. Different naming conventions. If your business logic has to understand both shapes, you've coupled your system to two external contracts simultaneously. That's twice the places that break."

---

## 2.5 Validators as Guardrails

**Validators on external models are not about formatting — they're about catching impossible data before it poisons your system.** (Using `field_validator` from Week 3, same syntax, different purpose.)

```python
from pydantic import field_validator

class OpenWeatherMain(BaseModel):
    model_config = ConfigDict(extra="ignore")
    
    temp: float
    humidity: int | None = None
    pressure: float | None = None
    
    @field_validator("temp")
    @classmethod
    def temperature_must_be_physically_possible(cls, v: float) -> float:
        """Reject clearly impossible temperatures.
        OpenWeatherMap uses Kelvin. Valid range: ~180K to ~330K on Earth.
        This catches unit changes, garbage values, and sentinel codes."""
        if v < 100 or v > 400:
            raise ValueError(
                f"Temperature {v}K is outside physically plausible range. "
                f"API may have changed units or sent a sentinel value."
            )
        return v
    
    @field_validator("humidity")
    @classmethod
    def humidity_must_be_percentage(cls, v: int | None) -> int | None:
        """Humidity must be 0-100% or None."""
        if v is not None and not (0 <= v <= 100):
            return None  # Treat out-of-range as missing, don't crash
        return v
```

**Notice the two different strategies:**

```
┌─────────────────────────────────────────────────────────────────┐
│              VALIDATOR STRATEGIES FOR EXTERNAL DATA             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REJECT (raise ValueError):                                     │
│  Use when the field is CRITICAL and a bad value means           │
│  the entire response is untrustworthy.                          │
│  → "If the temperature is -9999K, something is fundamentally   │
│     wrong. Reject the whole response."                          │
│                                                                 │
│  COERCE TO NONE (return None):                                  │
│  Use when the field is SUPPLEMENTARY and a bad value            │
│  shouldn't torpedo the whole record.                            │
│  → "If humidity is 999, treat it as missing. We can still      │
│     use the temperature and description."                       │
│                                                                 │
│  CLAMP / TRANSFORM:                                             │
│  Use when you know the API has a quirk you can correct.         │
│  → "Provider sends -1 for 'unknown'. Map to None."             │
│  → "Provider sends 0-based priority. Add 1."                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Remember from our demo (Section 1.1):** Scenario 3 was the silent killer — the API changed from Kelvin to Celsius, and our code computed -253°C without raising an error. A temperature range validator catches this immediately. The value `20.0` fails the `v < 100` check for Kelvin, and you know something changed.

---

# PART 3: DATA NORMALIZATION

## 3.1 N Suppliers, One Warehouse Format

**You now have two providers, each with their own external models. Both return weather data. Your system needs ONE consistent format.**

**Without normalization, your business logic becomes this:**

```python
# ❌ This is what happens without an internal model:

async def get_temperature(city: str, provider: str) -> float:
    if provider == "openweathermap":
        data = await fetch_openweather(city)
        return data["main"]["temp"] - 273.15  # Kelvin → Celsius
    elif provider == "weatherapi":
        data = await fetch_weatherapi(city)
        return data["current"]["temp_c"]      # Already Celsius
    elif provider == "tomorrow_io":
        data = await fetch_tomorrow(city)
        return data["data"]["values"]["temperature"]  # Celsius
    # ...every function, every endpoint, every test
    # must know every provider's shape. Forever.
```

**This doesn't scale.** Adding a third provider means touching every function that uses weather data. Removing a provider means searching your entire codebase. Changing a provider's field name means changes in 20 files.

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE MESS WITHOUT NORMALIZATION               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌───────────┐                                │
│  Provider A ──────▶│           │                                │
│  (Kelvin, m/s)     │  Your     │                                │
│                    │  Business │                                │
│  Provider B ──────▶│  Logic    │  ← Must understand ALL formats │
│  (Celsius, km/h)   │           │  ← Conditionals everywhere     │
│                    │  (MESS)   │  ← Every new provider = more   │
│  Provider C ──────▶│           │     if/elif branches           │
│  (Fahrenheit, mph) │           │                                │
│                    └───────────┘                                │
│                                                                 │
│                                                                 │
│                    THE CLEAN VERSION                             │
│                                                                 │
│  Provider A ──▶ Adapter A ──┐                                   │
│                             │     ┌───────────┐                │
│  Provider B ──▶ Adapter B ──┼────▶│  Your     │                │
│                             │     │  Business │ ← Knows ONE     │
│  Provider C ──▶ Adapter C ──┘     │  Logic    │   format only   │
│                                   └───────────┘                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 The Internal Model

**Define ONE model that represents YOUR understanding of weather data. This is YOUR barcode label — the one every shelf in your warehouse uses.**

```python
from datetime import datetime, UTC
from pydantic import BaseModel, Field

class WeatherData(BaseModel):
    """Internal weather representation.
    ALL providers normalize to this shape.
    This model is the CONTRACT for the rest of your application.
    
    Naming: your conventions, not any provider's.
    Units: always Celsius, always meters/second.
    """
    city: str
    country: str | None = None
    temperature_celsius: float = Field(description="Always in Celsius")
    humidity_percent: int | None = Field(
        default=None, ge=0, le=100
    )
    description: str
    wind_speed_ms: float = Field(
        description="Always in meters/second"
    )
    source: str            # Which provider this came from
    fetched_at: datetime   # When we actually fetched from the API
```

**Design decisions in this model:**

```
┌─────────────────────────────────────────────────────────────────┐
│              INTERNAL MODEL DESIGN CHOICES                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  temperature_celsius  (not "temp", not "temp_c")                │
│  → Explicit unit in the name. No ambiguity, ever.              │
│                                                                 │
│  wind_speed_ms  (not "wind_speed", not "wind_kph")              │
│  → One canonical unit. Adapters convert FROM provider units.    │
│                                                                 │
│  source: str                                                    │
│  → Tracks provenance. Which API gave us this data?              │
│    Critical for debugging, auditing, fallback decisions.        │
│                                                                 │
│  fetched_at: datetime                                           │
│  → When we fetched, NOT when the provider measured.             │
│    This drives your staleness checks later (Part 5).            │
│                                                                 │
│  No provider-specific fields                                    │
│  → No "openweather_id" or "weatherapi_code". Those belong      │
│    in the external models, not here.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 The Adapter Pattern

**Each provider gets an adapter: a class that knows how to fetch from THAT provider's API and return YOUR internal model.**

**First, define the contract.** Using `Protocol` from `typing` — it defines an interface through structure, not inheritance. Any class with matching method signatures satisfies the protocol automatically:

```python
from typing import Protocol

class WeatherProvider(Protocol):
    """Any class that implements these methods qualifies as a WeatherProvider.
    No inheritance required — just match the method signatures."""
    
    @property
    def source_name(self) -> str: ...
    
    async def fetch_weather(self, city: str) -> WeatherData: ...
```

**Now build the adapters. Provider A — OpenWeatherMap:**

```python
class OpenWeatherAdapter:
    """Fetches from OpenWeatherMap. Returns internal WeatherData.
    OpenWeatherMap-specific details are TRAPPED inside this class."""
    
    source_name: str = "openweathermap"
    
    def __init__(self, client: httpx.AsyncClient, api_key: str) -> None:
        self._client = client
        self._api_key = api_key
        self._base_url = "https://api.openweathermap.org/data/2.5"
    
    async def fetch_weather(self, city: str) -> WeatherData:
        resp = await self._client.get(
            f"{self._base_url}/weather",
            params={"q": city, "appid": self._api_key},
        )
        resp.raise_for_status()
        
        # Parse into external model (validated, typed)
        external = OpenWeatherResponse.model_validate(resp.json())
        
        # Transform to internal model
        return self._to_internal(external)
    
    def _to_internal(self, ext: OpenWeatherResponse) -> WeatherData:
        """The translation logic. All provider-specific quirks live HERE."""
        return WeatherData(
            city=ext.name,
            country=None,
            temperature_celsius=round(ext.main.temp - 273.15, 1),  # Kelvin → Celsius
            humidity_percent=ext.main.humidity,
            description=(
                ext.weather[0].description if ext.weather else "No description"
            ),
            wind_speed_ms=ext.wind.speed if ext.wind else 0.0,     # Already m/s
            source=self.source_name,
            fetched_at=datetime.now(UTC),
        )
```

**Provider B — WeatherAPI.com:**

```python
class WeatherAPIAdapter:
    """Fetches from WeatherAPI.com. Returns the same internal WeatherData.
    Completely different API shape, completely same output."""
    
    source_name: str = "weatherapi"
    
    def __init__(self, client: httpx.AsyncClient, api_key: str) -> None:
        self._client = client
        self._api_key = api_key
        self._base_url = "https://api.weatherapi.com/v1"
    
    async def fetch_weather(self, city: str) -> WeatherData:
        resp = await self._client.get(
            f"{self._base_url}/current.json",
            params={"q": city, "key": self._api_key},
        )
        resp.raise_for_status()
        
        external = WeatherAPIResponse.model_validate(resp.json())
        return self._to_internal(external)
    
    def _to_internal(self, ext: WeatherAPIResponse) -> WeatherData:
        return WeatherData(
            city=ext.location.name,
            country=ext.location.country,
            temperature_celsius=ext.current.temp_c,                # Already Celsius
            humidity_percent=ext.current.humidity,
            description=ext.current.condition.text,
            wind_speed_ms=round(ext.current.wind_kph / 3.6, 1),   # km/h → m/s
            source=self.source_name,
            fetched_at=datetime.now(UTC),
        )
```

**The power of this design:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  From the rest of your system's perspective:                    │
│                                                                 │
│  async def show_weather(                                        │
│      provider: WeatherProvider, city: str                       │
│  ) -> WeatherData:                                              │
│      return await provider.fetch_weather(city)                  │
│                                                                 │
│  This function works with ANY provider. It does not know        │
│  or care whether data came from OpenWeatherMap or WeatherAPI.   │
│  It never will. It just gets WeatherData.                       │
│                                                                 │
│  Adding a THIRD provider:                                       │
│  ├─ Write new external models (their schema)                    │
│  ├─ Write new adapter (their format → your internal model)      │
│  ├─ Register it in your dependency injection                    │
│  └─ ZERO changes to business logic, endpoints, or tests         │
│                                                                 │
│  Removing a provider:                                           │
│  ├─ Delete the adapter and external models                      │
│  └─ ZERO changes anywhere else                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 The Complete Transformation Pipeline

**Here's the full flow, end to end:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE TRANSFORMATION PIPELINE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. HTTP REQUEST            (httpx — Lecture 1)                  │
│     │                                                           │
│     ▼                                                           │
│  2. RAW DICT                (resp.json() → untyped dict)        │
│     │                                                           │
│     ▼                                                           │
│  3. EXTERNAL MODEL          (model_validate → typed, validated) │
│     │                       Shaped like THEIR API               │
│     ▼                                                           │
│  4. ADAPTER._to_internal()  (unit conversion, field mapping)    │
│     │                                                           │
│     ▼                                                           │
│  5. INTERNAL MODEL          (WeatherData — YOUR contract)       │
│     │                       Shaped like YOUR domain             │
│     ▼                                                           │
│  6. DATABASE / RESPONSE     (SQLAlchemy model or API response)  │
│                                                                 │
│                                                                 │
│  Failure at step 2: HTTP error        → retry/circuit breaker   │
│  Failure at step 3: Schema mismatch   → ValidationError → log  │
│  Failure at step 4: Bad data values   → adapter handles it      │
│  Each layer catches problems at the right level.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Using multiple providers with fallback:**

```python
async def get_weather_with_fallback(
    providers: list[WeatherProvider],
    city: str,
) -> WeatherData | None:
    """Try each provider in order. Return first success."""
    
    for provider in providers:
        try:
            return await provider.fetch_weather(city)
        except httpx.HTTPStatusError as e:
            logger.warning(
                f"{provider.source_name} returned {e.response.status_code} "
                f"for {city}"
            )
        except ValidationError as e:
            logger.error(
                f"{provider.source_name} schema mismatch for {city}: {e}"
            )
        except httpx.RequestError as e:
            logger.warning(f"{provider.source_name} unreachable: {e}")
    
    # All providers failed
    logger.error(f"All providers failed for {city}")
    return None
```

The rest of your application calls `get_weather_with_fallback(providers, "London")` and gets back a `WeatherData` or `None`. It does not know how many providers exist, what their APIs look like, or which one succeeded.

---

## 3.5 Error Recovery in the Pipeline

**When parsing fails, you need two things: a graceful response AND the evidence to debug.**

```python
async def fetch_weather(self, city: str) -> WeatherData | None:
    """Fetch with full error recovery. Returns None on failure."""
    raw_data: dict | None = None
    
    try:
        resp = await self._client.get(
            f"{self._base_url}/weather",
            params={"q": city, "appid": self._api_key},
        )
        resp.raise_for_status()
        raw_data = resp.json()
        
        external = OpenWeatherResponse.model_validate(raw_data)
        return self._to_internal(external)
    
    except ValidationError as e:
        # Schema mismatch — the API probably changed
        logger.error(
            "Schema mismatch from %s for city=%s: %s",
            self.source_name, city, e.error_count(),
        )
        # STORE the raw response — you need it to update your external model
        if raw_data is not None:
            await self._store_failed_response(
                city=city,
                raw_response=raw_data,
                error=str(e),
            )
        return None
    
    except httpx.HTTPStatusError as e:
        logger.warning(
            "%s returned HTTP %d for city=%s",
            self.source_name, e.response.status_code, city,
        )
        return None
```

**The critical detail: store failed raw responses.**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY STORE FAILED RAW RESPONSES?                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Monday: Your adapter works fine.                               │
│  Tuesday: ValidationError on every request.                     │
│                                                                 │
│  Without stored raw response:                                   │
│  → "Something changed. What? I have no idea. The error says     │
│     'field required' but I need to see what they're actually    │
│     sending now to fix my external model."                      │
│                                                                 │
│  With stored raw response:                                      │
│  → You look at the stored JSON. You see they renamed "wind"     │
│     to "wind_data". You update OpenWeatherWind, deploy, done.   │
│                                                                 │
│  Storage options:                                               │
│  ├─ Log it (structured logging with the full JSON)              │
│  ├─ Write to a "failed_responses" table (JSONB column)          │
│  └─ Write to a file/S3 bucket for analysis                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: PERSISTENCE

## 4.1 Why Store External Data Locally?

**Your warehouse has shelves for a reason. You don't make customers wait at the loading dock for a shipment to arrive.**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY STORE EXTERNAL DATA LOCALLY?                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AVAILABILITY                                                   │
│  Their API is down. Yours isn't. Serve stale data.              │
│  "OpenWeatherMap is experiencing an outage" shouldn't mean      │
│  YOUR users see a 500 error.                                    │
│                                                                 │
│  PERFORMANCE                                                    │
│  Database query: ~1–5ms                                         │
│  External API call: ~200–2000ms                                 │
│  That's a 100x difference your users feel on every request.     │
│                                                                 │
│  RATE LIMITS                                                    │
│  100 users request London weather in 1 minute.                  │
│  Without local storage: 100 API calls (rate limit hit).         │
│  With local storage: 1 API call + 99 database queries.          │
│                                                                 │
│  QUERYABILITY                                                   │
│  Can't JOIN your users table with an external API.              │
│  Can't filter "cities where temp > 30" across an HTTP call.     │
│  Local data is SQL-queryable, indexable, joinable.              │
│                                                                 │
│  HISTORY                                                        │
│  External APIs give you NOW. Your database gives you OVER TIME. │
│  "What was the temperature yesterday?" — the API won't tell     │
│  you. Your database will.                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Designing the Storage Model

**Your SQLAlchemy model for external data needs columns that regular domain entities don't: metadata about WHERE the data came from and WHEN you last fetched it.**

```python
from sqlalchemy import String, Float, Integer, UniqueConstraint, func
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column
from datetime import datetime

class WeatherRecord(Base):
    __tablename__ = "weather_records"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    
    # --- Domain data (from your internal model) ---
    city: Mapped[str] = mapped_column(String(100), index=True)
    temperature_celsius: Mapped[float]
    humidity_percent: Mapped[int | None]
    description: Mapped[str] = mapped_column(String(200))
    wind_speed_ms: Mapped[float]
    
    # --- External data metadata (NOT on your own entities) ---
    source: Mapped[str] = mapped_column(
        String(50),
        comment="Which provider: 'openweathermap', 'weatherapi', etc."
    )
    fetched_at: Mapped[datetime] = mapped_column(
        comment="When we fetched from the external API"
    )
    raw_response: Mapped[dict | None] = mapped_column(
        JSONB, nullable=True,
        comment="Original API response for debugging schema changes"
    )
    
    # --- Standard timestamps ---
    created_at: Mapped[datetime] = mapped_column(
        server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        server_default=func.now(), onupdate=func.now()
    )
    
    # One record per city per source
    __table_args__ = (
        UniqueConstraint("city", "source", name="uq_weather_city_source"),
    )
```

**The metadata columns explained:**

```
┌─────────────────────────────────────────────────────────────────┐
│              METADATA COLUMNS FOR EXTERNAL DATA                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  source                                                         │
│  ──────                                                         │
│  Which provider gave us this data. You might have weather       │
│  for London from two different APIs. This tells them apart.     │
│                                                                 │
│  fetched_at                                                     │
│  ─────────                                                      │
│  When YOUR system fetched this from the external API.           │
│  Different from created_at (when the row was inserted) and      │
│  different from the provider's measurement time.                │
│  Used for staleness checks: "Is this data too old?"             │
│                                                                 │
│  raw_response (JSONB — Week 5)                                  │
│  ────────────                                                   │
│  The original API response stored as JSON. This is your         │
│  insurance policy. When parsing breaks, you look here to see    │
│  what the API is actually sending now.                          │
│                                                                 │
│  UniqueConstraint("city", "source")                             │
│  ─────────────────────────────────                              │
│  One row per city per provider. If you fetch London from        │
│  OpenWeatherMap, it updates the SAME row (upsert, next section).│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Upsert: Create or Update in One Query

**When you sync external data, you need to handle two cases: the record doesn't exist yet (INSERT) or it does and needs updating (UPDATE). PostgreSQL handles both in one operation.**

**The SQL concept:**

```sql
-- Plain English: "Insert this weather data. But if a record for
-- this city+source already exists, update it instead."

INSERT INTO weather_records (city, source, temperature_celsius, ...)
VALUES ('London', 'openweathermap', 20.0, ...)
ON CONFLICT (city, source) DO UPDATE SET
    temperature_celsius = EXCLUDED.temperature_celsius,
    fetched_at = EXCLUDED.fetched_at,
    updated_at = NOW();
```

`EXCLUDED` is a special PostgreSQL keyword — it refers to the values that *would have been inserted* if there was no conflict. Think of it as "the new data."

**In SQLAlchemy, using the PostgreSQL-specific `insert`:**

```python
from sqlalchemy.dialects.postgresql import insert as pg_insert

async def upsert_weather(
    session: AsyncSession,
    weather: WeatherData,
    raw_response: dict | None = None,
) -> None:
    """Insert new weather record or update existing one.
    Uses PostgreSQL ON CONFLICT for atomic upsert."""
    
    values = {
        "city": weather.city,
        "source": weather.source,
        "temperature_celsius": weather.temperature_celsius,
        "humidity_percent": weather.humidity_percent,
        "description": weather.description,
        "wind_speed_ms": weather.wind_speed_ms,
        "fetched_at": weather.fetched_at,
        "raw_response": raw_response,
    }
    
    stmt = pg_insert(WeatherRecord).values(**values)
    
    stmt = stmt.on_conflict_do_update(
        constraint="uq_weather_city_source",  # The named unique constraint
        set_={
            # Update all data columns with the new values
            "temperature_celsius": stmt.excluded.temperature_celsius,
            "humidity_percent": stmt.excluded.humidity_percent,
            "description": stmt.excluded.description,
            "wind_speed_ms": stmt.excluded.wind_speed_ms,
            "fetched_at": stmt.excluded.fetched_at,
            "raw_response": stmt.excluded.raw_response,
            "updated_at": func.now(),
        },
    )
    
    await session.execute(stmt)
    await session.commit()
```

**Why upsert instead of check-then-insert?**

```python
# ❌ Race condition — DON'T do this:
existing = await session.get(WeatherRecord, ...)
if existing:
    existing.temperature_celsius = weather.temperature_celsius  # UPDATE
else:
    session.add(WeatherRecord(...))                             # INSERT
# Two concurrent requests for the same city can BOTH see "no existing"
# and BOTH try to INSERT → IntegrityError on the unique constraint

# ✅ Upsert is atomic — ONE query, no race condition
stmt = pg_insert(WeatherRecord).values(...).on_conflict_do_update(...)
# The database handles the check-and-insert/update as one atomic operation
```

**`stmt.excluded` explained:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    stmt.excluded                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INSERT INTO weather_records VALUES ('London', 20.0, ...)       │
│                                       ▲                         │
│  ON CONFLICT DO UPDATE SET            │                         │
│     temperature = EXCLUDED.temperature│                         │
│                            │          │                         │
│                            └──────────┘                         │
│                   "The row that WOULD have been inserted"       │
│                                                                 │
│  In SQLAlchemy:                                                 │
│     stmt.excluded.temperature_celsius                           │
│     → references the temperature_celsius from the VALUES(...)   │
│       clause — the NEW data you're trying to store              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Sync Strategies

**Not all external data should be synced the same way. Choose your strategy based on the data and the API.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      SYNC STRATEGIES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ON-DEMAND + CACHE                                              │
│  ─────────────────                                              │
│  Fetch when a user requests it. Cache for a TTL.                │
│  Next request within TTL → serve from database.                 │
│  After TTL expires → fetch again.                               │
│                                                                 │
│  ✅ No wasted API calls (only fetch what's requested)           │
│  ✅ Simple to implement                                         │
│  ❌ First request after expiry is slow                          │
│  Use when: Unpredictable access patterns (weather for any city) │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  SCHEDULED FULL REFRESH                                         │
│  ──────────────────────                                         │
│  Periodically fetch ALL data and replace in database.           │
│  "Every 30 minutes, sync all tracked cities."                   │
│                                                                 │
│  ✅ Data is always fresh when users request it                  │
│  ✅ Predictable API usage                                       │
│  ❌ Expensive for large datasets                                │
│  ❌ Wastes calls on data nobody is requesting                   │
│  Use when: Small, fixed dataset (e.g., 50 monitored cities)    │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  INCREMENTAL / DELTA SYNC                                       │
│  ────────────────────────                                       │
│  Only fetch records that changed since last sync.               │
│  "Give me everything modified after 2024-01-15T10:00:00Z."     │
│                                                                 │
│  ✅ Efficient for large datasets                                │
│  ❌ Requires API support (modified_since, cursor pagination)    │
│  Use when: Large dataset + API supports change tracking         │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  EVENT-DRIVEN (Webhooks)                                        │
│  ────────────────────────                                       │
│  The provider tells YOU when data changes.                      │
│  You built a webhook endpoint in Lecture 2 — this is where      │
│  it connects.                                                   │
│                                                                 │
│  ✅ Real-time updates, no polling                               │
│  ❌ Not all providers offer webhooks                            │
│  ❌ Requires a publicly reachable endpoint                     │
│  Use when: Provider supports webhooks (e.g., GitHub, Stripe)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Tracking freshness — the `is_stale` check:**

```python
from datetime import datetime, timedelta, UTC

def is_stale(record: WeatherRecord, max_age_minutes: int = 30) -> bool:
    """Check if a record is too old to serve without refresh."""
    age = datetime.now(UTC) - record.fetched_at
    return age > timedelta(minutes=max_age_minutes)
```

**What `max_age_minutes` should you use?** That depends on the domain, not the technology:

```
┌─────────────────────────────────────────────────────────────────┐
│                  STALENESS BY DOMAIN                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Weather data            →  30–60 minutes                       │
│  Exchange rates          →  5–15 minutes                        │
│  Stock prices            →  Seconds (probably don't cache)      │
│  Country codes / ISO     →  24 hours (rarely changes)           │
│  GitHub repository info  →  5–30 minutes                        │
│  User profile from OAuth →  1 hour                              │
│                                                                 │
│  Ask: "If a user sees data THIS old, do they care?"             │
│  Weather from 30 min ago? Fine. Stock price from 30 min ago?    │
│  Not fine.                                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: BACKGROUND REFRESH

## 5.1 The Stale Data Problem

**A user requests weather for London. Your database has a record, but it was fetched 45 minutes ago. What do you do?**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THREE OPTIONS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPTION A: Return stale data immediately                        │
│  ────────────────────────────────────────                       │
│  Response time: ~5ms (database query)                           │
│  Data accuracy: 45 minutes old                                  │
│  User experience: Fast, but possibly outdated                   │
│                                                                 │
│  OPTION B: Fetch fresh data, then respond                       │
│  ────────────────────────────────────────                       │
│  Response time: ~500–2000ms (external API call)                 │
│  Data accuracy: Current                                         │
│  User experience: Slow. User waits for external API.            │
│                                                                 │
│  OPTION C: Return stale data AND refresh in the background  ✅  │
│  ────────────────────────────────────────────────────────────   │
│  Response time: ~5ms (serve the stale record now)               │
│  Data accuracy: 45 min old NOW, fresh on NEXT request           │
│  User experience: Fast! And next request gets fresh data.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Option C is the "best of both worlds" pattern.** The user gets a fast response now. In the background, your system fetches fresh data and updates the database. The NEXT request gets fresh data with a fast database query.

> "It's like a warehouse that ships what's on the shelf immediately, then places a reorder so the shelf is restocked for the next customer."

---

## 5.2 FastAPI BackgroundTasks

**FastAPI provides `BackgroundTasks` — a way to run work AFTER the response has been sent to the client.** The client doesn't wait.

```python
from fastapi import BackgroundTasks, APIRouter, Depends

router = APIRouter()

async def refresh_weather_in_background(
    city: str,
    providers: list[WeatherProvider],
) -> None:
    """This runs AFTER the HTTP response is sent.
    
    IMPORTANT: This function needs its OWN database session.
    The route handler's session closes when the response is sent.
    """
    async with async_session_factory() as session:
        for provider in providers:
            try:
                data = await provider.fetch_weather(city)
                raw = None  # could also store raw response
                await upsert_weather(session, data, raw_response=raw)
                logger.info(
                    "Background refresh succeeded for %s from %s",
                    city, provider.source_name,
                )
                return  # Success — no need to try other providers
            except Exception as e:
                logger.warning(
                    "Background refresh failed for %s from %s: %s",
                    city, provider.source_name, e,
                )
                continue
        
        logger.error("Background refresh failed for %s from all providers", city)
```

**The route handler that uses it:**

```python
@router.get("/weather/{city}")
async def get_weather(
    city: str,
    background_tasks: BackgroundTasks,
    session: AsyncSession = Depends(get_session),
    providers: list[WeatherProvider] = Depends(get_providers),
) -> WeatherData:
    # Step 1: Check database
    record = await get_weather_from_db(session, city)
    
    if record is not None and not is_stale(record, max_age_minutes=30):
        # Fresh data in DB — serve it directly
        return record_to_weather_data(record)
    
    if record is not None and is_stale(record, max_age_minutes=30):
        # Stale data — serve it NOW, refresh in background
        background_tasks.add_task(
            refresh_weather_in_background, city, providers
        )
        return record_to_weather_data(record)  # Return stale, don't wait
    
    # No data at all — must fetch synchronously (first request ever)
    for provider in providers:
        try:
            data = await provider.fetch_weather(city)
            await upsert_weather(session, data)
            return data
        except Exception:
            continue
    
    raise HTTPException(status_code=502, detail="All weather providers failed")
```

**The session gotcha — why background tasks create their own session:**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE SESSION LIFECYCLE GOTCHA                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REQUEST LIFECYCLE:                                             │
│                                                                 │
│  Request arrives                                                │
│    │                                                            │
│    ▼                                                            │
│  Depends(get_session) → session opens                           │
│    │                                                            │
│    ▼                                                            │
│  Route handler runs                                             │
│    │                                                            │
│    ├─ Reads from DB using session     ✅ Session is open        │
│    ├─ Adds background task                                      │
│    │                                                            │
│    ▼                                                            │
│  Response sent to client                                        │
│    │                                                            │
│    ▼                                                            │
│  Session CLOSES (yield dependency teardown)                     │
│    │                                                            │
│    ▼                                                            │
│  Background task runs                                           │
│    │                                                            │
│    └─ Tries to use route's session → 💥 Session is closed!     │
│                                                                 │
│  SOLUTION: Background task creates its OWN session.             │
│                                                                 │
│  async def refresh_in_background(city: str):                    │
│      async with async_session_factory() as session:  ← NEW     │
│          # This session is independent of the request           │
│          await upsert_weather(session, data)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Refresh Patterns

**Two patterns for keeping data fresh:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    REFRESH PATTERNS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REFRESH-ON-READ (what we just built)                           │
│  ────────────────                                               │
│  Triggered by: A user requesting data                           │
│  Logic: "You asked for weather. I have stale data. I'll give    │
│   it to you now and update it in the background."               │
│                                                                 │
│  ✅ Only refreshes data that's actually being requested         │
│  ✅ No scheduled jobs to manage                                 │
│  ❌ First user after expiry always sees stale data              │
│  ❌ Burst of requests can trigger many redundant refreshes      │
│                                                                 │
│                                                                 │
│  SCHEDULED REFRESH (preview — you'll build this with Celery)    │
│  ─────────────────                                              │
│  Triggered by: A timer ("every 30 minutes")                     │
│  Logic: "It's 10:00. Time to refresh all tracked cities,        │
│   whether anyone is asking or not."                             │
│                                                                 │
│  ✅ Data is always fresh when requested                         │
│  ✅ Predictable API usage (no bursts)                           │
│  ❌ Wastes calls on data nobody requests                        │
│  ❌ Requires a task scheduler (Celery Beat — Week 11)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Preventing redundant refreshes:**

When 10 users request London weather within the same minute and the data is stale, you don't want 10 background refresh tasks hitting the same external API. A simple guard:

```python
# In-memory set of cities currently being refreshed
_refresh_in_progress: set[str] = set()

async def refresh_weather_in_background(
    city: str,
    providers: list[WeatherProvider],
) -> None:
    key = f"{city}"
    
    if key in _refresh_in_progress:
        return  # Another task is already refreshing this city
    
    _refresh_in_progress.add(key)
    try:
        async with async_session_factory() as session:
            data = await get_weather_with_fallback(providers, city)
            if data:
                await upsert_weather(session, data)
    finally:
        _refresh_in_progress.discard(key)
```

> "This is a simple in-memory guard. It works for a single server instance. For multiple server instances behind a load balancer, you'd use a Redis-based lock (Week 10)."

---

## 5.4 When BackgroundTasks Isn't Enough

**BackgroundTasks is simple and built-in. But it has real limitations:**

```
┌─────────────────────────────────────────────────────────────────┐
│              BACKGROUNDTASKS LIMITATIONS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ No retry on failure                                        │
│     Task fails? It's gone. No automatic retry.                 │
│                                                                 │
│  ❌ No persistence                                             │
│     Server restarts? All pending tasks are lost.               │
│                                                                 │
│  ❌ No scheduling                                              │
│     Can't say "run this every 30 minutes."                     │
│     Only triggered by incoming requests.                       │
│                                                                 │
│  ❌ No monitoring                                              │
│     Did the task succeed? How long did it take?                │
│     You only know if you check your logs manually.             │
│                                                                 │
│  ❌ Same process, same resources                               │
│     A slow background task competes for CPU and memory         │
│     with your request handlers.                                │
│                                                                 │
│                                                                 │
│  BackgroundTasks is fine for:                                   │
│  ├─ Sending a notification email after signup                  │
│  ├─ Refreshing one cached record after serving stale data      │
│  └─ Logging analytics events asynchronously                    │
│                                                                 │
│  For anything bigger — scheduled syncs, reliable retries,       │
│  distributed processing — you need Celery (Week 11).            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Choosing the right tool:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "Do I need this task to survive a server restart?"             │
│                                                                 │
│   NO  → BackgroundTasks is fine                                 │
│   YES → You need Celery (Week 11)                               │
│                                                                 │
│  "Do I need this to run on a schedule?"                         │
│                                                                 │
│   NO  → BackgroundTasks is fine                                 │
│   YES → You need Celery Beat (Week 11)                          │
│                                                                 │
│  "Do I need retry logic if it fails?"                           │
│                                                                 │
│   NO  → BackgroundTasks is fine                                 │
│   YES → You need Celery with autoretry (Week 11)               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│           DATA TRANSFORMATION QUICK REFERENCE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PARSE EXTERNAL DATA:                                           │
│      external = ProviderResponse.model_validate(resp.json())    │
│                                                                 │
│  EXTERNAL MODEL TEMPLATE:                                       │
│      class ProviderResponse(BaseModel):                         │
│          model_config = ConfigDict(extra="ignore")              │
│          required_field: str                                    │
│          optional_field: int | None = None                      │
│                                                                 │
│  TRANSFORM TO INTERNAL:                                         │
│      internal = WeatherData(                                    │
│          city=external.location.name,                           │
│          temperature_celsius=external.current.temp_c,           │
│          source="provider_name",                                │
│          fetched_at=datetime.now(UTC),                           │
│      )                                                          │
│                                                                 │
│  UPSERT TO DATABASE:                                            │
│      stmt = pg_insert(Model).values(**data)                     │
│      stmt = stmt.on_conflict_do_update(                         │
│          constraint="unique_constraint_name",                   │
│          set_={"col": stmt.excluded.col, ...},                  │
│      )                                                          │
│                                                                 │
│  CHECK STALENESS:                                               │
│      age = datetime.now(UTC) - record.fetched_at                │
│      is_stale = age > timedelta(minutes=30)                     │
│                                                                 │
│  BACKGROUND REFRESH:                                            │
│      background_tasks.add_task(refresh_fn, city, providers)     │
│      (Remember: background task needs its OWN db session)       │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ data["field"]["nested"]  → Use Pydantic external model  │
│      ❌ Same model for all       → Separate external & internal │
│      ❌ extra="forbid" on ext.   → Use extra="ignore"          │
│      ❌ No raw_response stored   → Can't debug schema changes  │
│      ❌ Route session in bg task → Create a new session         │
│      ❌ No source/fetched_at     → Can't track provenance      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  THE WAREHOUSE RECEIVING DOCK                                   │
│                                                                 │
│  ┌──────────┐    ┌───────────────┐    ┌──────────┐              │
│  │ Supplier │    │  Receiving    │    │ Warehouse│              │
│  │(Ext. API)│───▶│  Dock         │───▶│ Shelves  │              │
│  └──────────┘    │ (Transform)   │    │(Database)│              │
│                  └───────────────┘    └──────────┘              │
│                                                                 │
│  1. Shipment arrives (httpx gets a response)                    │
│  2. Inspect with checklist (parse into external Pydantic model) │
│  3. Reject if damaged (ValidationError → log + skip)            │
│  4. Relabel to your standard (adapter → internal model)         │
│  5. Store on your shelves (upsert to PostgreSQL)                │
│  6. Check sell-by dates (is_stale check on fetched_at)          │
│  7. Reorder when low (BackgroundTasks refresh)                  │
│                                                                 │
│  THREE RULES:                                                   │
│  ├─ NEVER store foreign-labeled boxes on your shelves           │
│  │  (never save external model shapes directly to your DB)      │
│  ├─ NEVER make warehouse staff learn every supplier's language  │
│  │  (never let business logic parse provider-specific formats)  │
│  └─ ALWAYS have one standard labeling system                    │
│     (one internal model that all providers normalize to)        │
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
│  WEEK 8 PROJECT:                                                │
│  └─ Third-Party Integration Service                             │
│     Apply EVERYTHING from this lecture: external models for     │
│     2-3 real APIs (GitHub, OpenWeatherMap, etc.), adapters      │
│     for each, normalization to internal models, PostgreSQL      │
│     storage with upserts, background refresh.                   │
│                                                                 │
│  WEEK 9 (Authentication):                                       │
│  └─ OAuth providers are external APIs too                       │
│     User profile data from Google/GitHub needs the same         │
│     external model → internal model transformation              │
│                                                                 │
│  WEEK 10 (Redis & Caching):                                     │
│  └─ Cache external API responses in Redis                       │
│     Faster than DB for frequently accessed external data        │
│     Redis TTL replaces your manual is_stale() check             │
│                                                                 │
│  WEEK 11 (Celery & Background Jobs):                            │
│  └─ Replace BackgroundTasks with scheduled Celery jobs          │
│     Reliable retries, persistent queue, monitoring              │
│     "Sync all tracked cities every 30 minutes" — Celery Beat   │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ Integration patterns at production scale                    │
│     Email services, file storage, search — all external         │
│     services that need the same trust-boundary discipline       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```