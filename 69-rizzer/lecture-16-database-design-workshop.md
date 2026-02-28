# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CHAOS FIRST, ORDER SECOND                                      │
│  ─────────────────────────                                      │
│  Students see a real requirements doc. They try to put it       │
│  all in one table. They feel the pain. THEN we design.          │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Database design is architecture. You draw blueprints before    │
│  pouring concrete. Every design decision maps to: "What         │
│  happens when this building needs a new wing in 6 months?"      │
│                                                                 │
│  DECISION-DRIVEN, NOT RULE-DRIVEN                               │
│  ────────────────────────────────                               │
│  There is no single "correct" schema. Every choice is a         │
│  tradeoff. Students must learn to ARGUE for their design,       │
│  not memorize patterns blindly.                                 │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Lecture 1 → Normalization is the WHY behind splitting tables   │
│  Lecture 2 → JOINs are HOW you reconnect what you split         │
│  Lecture 3 → PostgreSQL types/features inform WHAT you store    │
│  Week 3-4 → Pydantic models mirror table schemas closely       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                  DATABASE DESIGN WORKSHOP                       │
│                     (3-4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: FROM REQUIREMENTS TO ENTITIES (45 min)                 │
│  ├─ 1.1 The Business Brief (QuickBite Food Delivery)            │
│  ├─ 1.2 The God Table (The Naive Approach)                      │
│  ├─ 1.3 Extracting Entities (The Noun Technique)                │
│  └─ 1.4 The Blueprint Mindset                                   │
│                                                                 │
│  PART 2: RELATIONSHIP PATTERNS (50 min)                         │
│  ├─ 2.1 One-to-Many: The Parent-Child Pattern                   │
│  ├─ 2.2 Many-to-Many: When One FK Isn't Enough                  │
│  ├─ 2.3 Junction Tables: The Bridge Entity                      │
│  └─ 2.4 Self-Referential Relationships                          │
│                                                                 │
│  PART 3: PRIMARY KEY STRATEGIES (30 min)                        │
│  ├─ 3.1 Serial IDs: Simple and Sequential                       │
│  ├─ 3.2 UUIDs: Globally Unique                                  │
│  ├─ 3.3 The Decision Framework                                  │
│  └─ 3.4 Composite Keys: When One Column Isn't Enough            │
│                                                                 │
│  PART 4: DATA LIFECYCLE PATTERNS (30 min)                       │
│  ├─ 4.1 Hard Deletes: Gone Forever                              │
│  ├─ 4.2 Soft Deletes: Hidden but Preserved                      │
│  ├─ 4.3 The Decision Framework                                  │
│  └─ 4.4 Timestamps: Tracking When Things Happen                 │
│                                                                 │
│  PART 5: THE COMPLETE DESIGN PROCESS (45 min)                   │
│  ├─ 5.1 The Step-by-Step Method                                 │
│  ├─ 5.2 QuickBite: Full Schema Walkthrough                      │
│  ├─ 5.3 Common Design Mistakes                                  │
│  └─ 5.4 The Design Review Checklist                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: FROM REQUIREMENTS TO ENTITIES

## 1.1 The Business Brief (QuickBite Food Delivery)

**You are a backend engineer. Your PM drops this in Slack on Monday morning:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    QUICKBITE — PRODUCT BRIEF                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  QuickBite is a food delivery platform.                         │
│                                                                 │
│  • Restaurants sign up and list their menu items.               │
│  • Each menu item belongs to a category (e.g., "Appetizers",    │
│    "Main Course", "Desserts"). Categories are per-restaurant    │
│    — each restaurant defines its own set.                       │
│  • Customers browse restaurants, add items to a cart,           │
│    and place orders.                                            │
│  • An order can include items from ONE restaurant only.         │
│  • Each order item records the quantity and the price at the    │
│    time of order (prices may change later).                     │
│  • Customers can leave a review (1-5 stars + text) on a        │
│    restaurant after completing an order.                        │
│  • A customer can only review a restaurant once.                │
│  • Restaurants have an owner (one user), but we might add       │
│    staff roles later.                                           │
│  • Menu items can be marked as unavailable without removing     │
│    them.                                                        │
│  • We need to track when everything was created and last        │
│    updated.                                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "You've used SQL for three lectures. You know CREATE TABLE, JOINs, data types, indexes. But right now — looking at this brief — how many tables do you need? What columns? What references what? That blank-page feeling? That's the design problem. SQL is the construction crew. This lecture teaches you to be the architect."

---

## 1.2 The God Table (The Naive Approach)

**Let's do what feels natural first. Shove everything into one table.**

```sql
-- ❌ THE GOD TABLE — everything in one place
CREATE TABLE everything (
    order_id        INTEGER,
    order_date      TIMESTAMP,
    customer_name   TEXT,
    customer_email  TEXT,
    customer_phone  TEXT,
    restaurant_name TEXT,
    restaurant_addr TEXT,
    owner_name      TEXT,
    owner_email     TEXT,
    category_name   TEXT,
    item_name       TEXT,
    item_price      NUMERIC,
    quantity        INTEGER,
    review_stars    INTEGER,
    review_text     TEXT
);
```

**Insert some data:**

```
order_id | customer_name | customer_email      | restaurant_name | item_name    | item_price | quantity | category_name
---------|---------------|---------------------|-----------------|--------------|------------|----------|-------------
1        | Alice         | alice@mail.com      | Burger Palace   | Classic Burger| 12.99     | 2        | Main Course
1        | Alice         | alice@mail.com      | Burger Palace   | Fries        | 4.99      | 1        | Sides
1        | Alice         | alice@mail.com      | Burger Palace   | Cola         | 2.99      | 2        | Drinks
2        | Bob           | bob@mail.com        | Burger Palace   | Classic Burger| 12.99     | 1        | Main Course
3        | Alice         | alice@mail.com      | Sushi Zen       | Salmon Roll  | 15.99     | 3        | Rolls
```

**Now ask the class:**

> "Alice ordered from Burger Palace. Her name and email appear THREE times — once per item. Bob also ordered from Burger Palace, so the restaurant name and address repeat again. What happens when Alice changes her email?"

**The problems explode:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  GOD TABLE PROBLEMS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. UPDATE ANOMALY                                              │
│  ─────────────────                                              │
│  Alice changes her email. You must update EVERY ROW she         │
│  appears in. Miss one? Now she has two different emails         │
│  in your database. Which is correct?                            │
│                                                                 │
│  2. INSERT ANOMALY                                              │
│  ─────────────────                                              │
│  A new restaurant signs up but has no orders yet. You can't     │
│  insert it — most columns would be NULL. What's the order_id    │
│  for a restaurant with no orders?                               │
│                                                                 │
│  3. DELETE ANOMALY                                              │
│  ─────────────────                                              │
│  Alice's only order gets cancelled and you delete those rows.   │
│  Now Alice the customer is GONE. Her account, her profile —     │
│  deleted, because her existence was tied to an order.           │
│                                                                 │
│  4. WASTED SPACE                                                │
│  ──────────────                                                 │
│  "Burger Palace" + its address is stored in every single row    │
│  of every order from that restaurant. Multiply by 10,000        │
│  orders. That's 10,000 copies of the same string.               │
│                                                                 │
│  5. NO INTEGRITY                                                │
│  ──────────────                                                 │
│  Nothing stops you from writing "Burger Palace" in one row      │
│  and "Burger Palce" in another. Typos become data corruption.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to Lecture 1:**

> "These are the exact anomalies normalization exists to solve. In Lecture 1, we talked about 1NF, 2NF, 3NF as rules. Now you're FEELING why those rules exist. Normalization isn't academic — it prevents the chaos you just saw."

---

## 1.3 Extracting Entities (The Noun Technique)

**Go back to the requirements. Underline every noun.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE NOUN TECHNIQUE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Read the requirements. Underline every NOUN.                   │
│  Each noun is a CANDIDATE entity (table).                       │
│                                                                 │
│  "[Restaurants] sign up and list their [menu items]."           │
│  "Each [menu item] belongs to a [category]."                    │
│  "[Customers] browse [restaurants], add [items] to a [cart],    │
│   and place [orders]."                                          │
│  "An [order] can include [items] from ONE [restaurant] only."   │
│  "[Customers] can leave a [review] on a [restaurant]."          │
│  "[Restaurants] have an [owner] (one [user])."                  │
│                                                                 │
│  Candidate entities:                                            │
│  ├─ Restaurant                                                  │
│  ├─ Menu Item                                                   │
│  ├─ Category                                                    │
│  ├─ Customer / User / Owner  ← same entity? probably.          │
│  ├─ Cart                     ← transient state, not a table?   │
│  ├─ Order                                                       │
│  ├─ Order Item               ← "each order item records..."    │
│  └─ Review                                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Not every noun becomes a table. Filter:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  NOUN FILTERING                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ASK FOR EACH NOUN:                                             │
│                                                                 │
│  1. Does it have its own attributes?                            │
│     "Category" has a name. → Probably a table (or a column).    │
│                                                                 │
│  2. Can there be multiple instances?                            │
│     Multiple restaurants, multiple orders → tables.             │
│                                                                 │
│  3. Does it need to exist independently?                        │
│     "Cart" is temporary UI state. It may not need a table.      │
│     (Or it might — depends on requirements. Ask your PM.)       │
│                                                                 │
│  4. Is it really just an attribute of something else?           │
│     "Owner" is a USER who owns a restaurant — it's a role,      │
│     not a separate entity.                                      │
│                                                                 │
│                                                                 │
│  FILTERED ENTITIES (for QuickBite v1):                          │
│  ├─ users           (customers, owners — one user table)        │
│  ├─ restaurants                                                 │
│  ├─ categories      (per-restaurant menu categories)            │
│  ├─ menu_items                                                  │
│  ├─ orders                                                      │
│  ├─ order_items     (line items within an order)                │
│  └─ reviews                                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now extract verbs — they tell you the relationships:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE VERB TECHNIQUE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Read the requirements. Identify every VERB connecting          │
│  two entities. The verb IS the relationship.                    │
│                                                                 │
│  "Restaurants LIST their menu items"                            │
│       └─ restaurant ──(has many)──▶ menu_items                  │
│                                                                 │
│  "Each menu item BELONGS TO a category"                         │
│       └─ category ──(has many)──▶ menu_items                    │
│                                                                 │
│  "Customers PLACE orders"                                       │
│       └─ user ──(has many)──▶ orders                            │
│                                                                 │
│  "An order INCLUDES items from ONE restaurant"                  │
│       └─ restaurant ──(has many)──▶ orders                      │
│       └─ order ──(has many)──▶ order_items                      │
│       └─ menu_item ──(referenced by)──▶ order_items             │
│                                                                 │
│  "Customers LEAVE a review ON a restaurant"                     │
│       └─ user ──(has many)──▶ reviews                           │
│       └─ restaurant ──(has many)──▶ reviews                     │
│                                                                 │
│  "Restaurants have an OWNER (one user)"                         │
│       └─ user ──(owns many)──▶ restaurants                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 The Blueprint Mindset

**Database design is architecture. Internalize this.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE BLUEPRINT MINDSET                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ARCHITECTURE                     DATABASE DESIGN               │
│  ────────────                     ───────────────               │
│  Blueprint                        Schema / ER Diagram           │
│  Rooms                            Tables                        │
│  Room purpose (kitchen, bedroom)  Table purpose (users, orders) │
│  Doors connecting rooms           Foreign keys                  │
│  Building codes / zoning          Normalization rules           │
│  Room numbers                     Primary keys                  │
│  Shared hallways                  Junction tables               │
│  "Storage room" vs "dumpster"     Soft delete vs hard delete    │
│  Security cameras                 Timestamps                    │
│                                                                 │
│                                                                 │
│  THE CARDINAL RULE:                                             │
│  ──────────────────                                             │
│  It is 100x cheaper to redraw a blueprint than to demolish      │
│  a wall after the building is finished.                         │
│                                                                 │
│  It is 100x cheaper to redesign a schema before data exists     │
│  than to migrate a production database with 10 million rows.    │
│                                                                 │
│  DESIGN FIRST. CODE SECOND.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The practical workflow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  DESIGN BEFORE CODE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  Requirements ──▶ Nouns/Verbs ──▶ ER Diagram ──▶ CREATE TABLE   │
│       │               │               │               │        │
│       │               │               │               │        │
│    (read)         (extract)       (draw)          (code)       │
│                                                                 │
│       ◀──────────── Review Loop ──────────────────┘            │
│                                                                 │
│  You will iterate. The first design is never final.             │
│  But starting with a diagram saves you from starting            │
│  with ALTER TABLE.                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 2: RELATIONSHIP PATTERNS

## 2.1 One-to-Many: The Parent-Child Pattern

**The most common relationship in any database. You'll use this everywhere.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  ONE-TO-MANY                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "One restaurant HAS MANY menu items."                          │
│  "One user HAS MANY orders."                                    │
│  "One order HAS MANY order items."                              │
│                                                                 │
│                                                                 │
│   ONE side                          MANY side                   │
│  ┌──────────────┐              ┌──────────────┐                 │
│  │ restaurants   │              │ menu_items   │                │
│  │──────────────│    1    ∞    │──────────────│                │
│  │ id        PK │─────────────▶│ id        PK │                │
│  │ name         │              │ restaurant_id │◀── FK          │
│  │ address      │              │ name          │                │
│  │ ...          │              │ price         │                │
│  └──────────────┘              └──────────────┘                 │
│                                                                 │
│  THE RULE:                                                      │
│  The foreign key goes on the MANY side.                         │
│  Each menu item points to ONE restaurant.                       │
│  Each restaurant is pointed to by MANY menu items.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why does the FK go on the MANY side? Think about it physically:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ FK on the ONE side — WHERE WOULD YOU PUT IT?                │
│                                                                 │
│  restaurants                                                    │
│  ──────────────────────────────                                 │
│  id │ name          │ menu_item_id ← ???                        │
│   1 │ Burger Palace │ 1? 2? 3? 47?   CAN'T FIT MULTIPLE!       │
│                                                                 │
│  You'd need a column for EACH menu item.                        │
│  How many columns? You don't know. Impossible.                  │
│                                                                 │
│                                                                 │
│  ✅ FK on the MANY side — ONE value per row. Clean.             │
│                                                                 │
│  menu_items                                                     │
│  ──────────────────────────────────────                         │
│  id │ restaurant_id │ name           │ price                    │
│   1 │ 1             │ Classic Burger │ 12.99                    │
│   2 │ 1             │ Fries          │  4.99                    │
│   3 │ 1             │ Cola           │  2.99                    │
│   4 │ 2             │ Salmon Roll    │ 15.99                    │
│                                                                 │
│  Each row has exactly ONE restaurant_id. Simple. Scalable.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In SQL (you already know this from Lecture 2, now applied to design):**

```sql
CREATE TABLE restaurants (
    id    BIGSERIAL PRIMARY KEY,
    name  TEXT NOT NULL,
    address TEXT NOT NULL
);

CREATE TABLE menu_items (
    id              BIGSERIAL PRIMARY KEY,
    restaurant_id   BIGINT NOT NULL REFERENCES restaurants(id),
    name            TEXT NOT NULL,
    price           NUMERIC(10, 2) NOT NULL,
    is_available    BOOLEAN NOT NULL DEFAULT true
);
```

**The REFERENCES keyword is your guardrail:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  REFERENTIAL INTEGRITY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REFERENCES restaurants(id) means:                              │
│                                                                 │
│  ✅ INSERT menu item with restaurant_id = 1     (if restaurant  │
│                                                  1 exists)      │
│  ❌ INSERT menu item with restaurant_id = 999   (if restaurant  │
│                                                  999 doesn't)   │
│     → ERROR: violates foreign key constraint                    │
│                                                                 │
│  ❌ DELETE restaurant 1 while it has menu items                  │
│     → ERROR: violates foreign key constraint                    │
│     (unless you specify ON DELETE CASCADE or SET NULL)           │
│                                                                 │
│  The database ENFORCES your relationships.                      │
│  No orphan rows. No dangling references.                        │
│  This is why we use foreign keys, not just matching IDs         │
│  by convention.                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**ON DELETE behavior — a design choice you must make:**

```sql
-- Option 1: RESTRICT (default) — prevent deletion if children exist
restaurant_id BIGINT NOT NULL REFERENCES restaurants(id)
-- DELETE FROM restaurants WHERE id = 1 → ERROR if menu items exist

-- Option 2: CASCADE — delete children automatically
restaurant_id BIGINT NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE
-- DELETE FROM restaurants WHERE id = 1 → also deletes all its menu items

-- Option 3: SET NULL — orphan the children
restaurant_id BIGINT REFERENCES restaurants(id) ON DELETE SET NULL
-- DELETE FROM restaurants WHERE id = 1 → menu_items.restaurant_id = NULL
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  ON DELETE: WHEN TO USE WHAT                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RESTRICT (default)                                             │
│  Use when: Children should NOT be destroyed with parent.        │
│  Example: Don't delete a user if they have orders.              │
│  Force explicit cleanup first.                                  │
│                                                                 │
│  CASCADE                                                        │
│  Use when: Children are meaningless without parent.             │
│  Example: Delete an order → delete its order_items.             │
│  An order item without an order makes no sense.                 │
│                                                                 │
│  SET NULL                                                       │
│  Use when: Children can exist independently.                    │
│  Example: Delete a category → menu items become uncategorized.  │
│  (Rare in practice. Usually means your model needs rethinking.) │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 Many-to-Many: When One FK Isn't Enough

**Some relationships go both directions in multiples.**

**Now ask the class:**

> "Let's think about orders and menu items. One order contains MANY menu items (burger + fries + cola). One menu item appears in MANY orders (the Classic Burger has been ordered 5,000 times). Where does the foreign key go?"

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE MANY-TO-MANY PROBLEM                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│   ┌──────────────┐                  ┌──────────────┐            │
│   │   orders     │      ∞    ∞      │  menu_items  │            │
│   │──────────────│  ◀───────────▶   │──────────────│            │
│   │ id           │                  │ id           │            │
│   │ user_id      │                  │ name         │            │
│   │ restaurant_id│                  │ price        │            │
│   └──────────────┘                  └──────────────┘            │
│                                                                 │
│                                                                 │
│   ❌ FK on orders? Which menu_item_id? There are MANY.          │
│   ❌ FK on menu_items? Which order_id? There are MANY.          │
│                                                                 │
│   Neither side can hold a single FK.                            │
│   You CANNOT represent many-to-many with two tables alone.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**This is where the junction table comes in.**

---

## 2.3 Junction Tables: The Bridge Entity

**A junction table sits BETWEEN two entities, holding FKs to both.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  JUNCTION TABLE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────┐       ┌──────────────┐       ┌──────────┐       │
│   │  orders  │  1  ∞ │ order_items  │ ∞  1  │menu_items│       │
│   │──────────│───────▶│──────────────│◀──────│──────────│       │
│   │ id    PK │       │ id        PK │       │ id    PK │       │
│   │ user_id  │       │ order_id  FK │       │ name     │       │
│   │ rest_id  │       │ menu_item_id │ FK    │ price    │       │
│   └──────────┘       │ quantity     │       └──────────┘       │
│                      │ unit_price   │                           │
│                      └──────────────┘                           │
│                                                                 │
│   The junction table BREAKS a many-to-many                      │
│   into TWO one-to-many relationships:                           │
│                                                                 │
│   orders ──(1:∞)──▶ order_items ◀──(∞:1)── menu_items          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In SQL:**

```sql
CREATE TABLE order_items (
    id            BIGSERIAL PRIMARY KEY,
    order_id      BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    menu_item_id  BIGINT NOT NULL REFERENCES menu_items(id),
    quantity      INTEGER NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(10, 2) NOT NULL
);
```

**Now ask the class:**

> "Why do we store `unit_price` on order_items instead of just JOINing to menu_items.price? Think about it. A burger costs $12.99 today. Next month it costs $14.99. What should Alice's order from last month show?"

**This is a critical design insight:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SNAPSHOT DATA                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Some values must be CAPTURED AT THE TIME OF THE EVENT,         │
│  not looked up dynamically.                                     │
│                                                                 │
│  ❌ BAD: Calculate order total by JOINing to menu_items.price   │
│     → Price changes break historical orders                     │
│     → Alice's $12.99 burger retroactively becomes $14.99        │
│     → Accounting is destroyed                                   │
│                                                                 │
│  ✅ GOOD: Store unit_price on order_items                       │
│     → Frozen at time of purchase                                │
│     → Historical data is accurate forever                       │
│     → Menu price changes don't affect past orders               │
│                                                                 │
│  This pattern applies everywhere:                               │
│  ├─ Order totals (snapshot item prices)                         │
│  ├─ Invoice addresses (snapshot at time of billing)             │
│  ├─ Shipping addresses (snapshot at time of shipment)           │
│  └─ Usernames in logs (snapshot at time of action)              │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  If the source value CAN change, and you need the               │
│  HISTORICAL value, snapshot it.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Junction tables often carry their own data:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  JUNCTION TABLE PAYLOADS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A "pure" junction table only has two FKs:                      │
│                                                                 │
│  user_favorites                                                 │
│  ─────────────────                                              │
│  user_id  FK  ──▶ users                                         │
│  restaurant_id FK ──▶ restaurants                               │
│  (maybe: favorited_at TIMESTAMP)                                │
│                                                                 │
│                                                                 │
│  A "rich" junction table carries relationship-specific data:    │
│                                                                 │
│  order_items                                                    │
│  ─────────────────                                              │
│  order_id     FK ──▶ orders                                     │
│  menu_item_id FK ──▶ menu_items                                 │
│  quantity        ← HOW MANY of this item in this order          │
│  unit_price      ← WHAT PRICE at the time of this order        │
│  special_notes   ← "no onions please" for this specific item   │
│                                                                 │
│  The payload data DESCRIBES THE RELATIONSHIP, not either        │
│  entity alone. "quantity" doesn't belong on orders              │
│  (which item?) or menu_items (which order?).                    │
│  It belongs on the LINK between them.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Naming conventions for junction tables:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  NAMING JUNCTION TABLES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPTION A: Combine both entity names                            │
│  ──────────────────────────────────                             │
│  users + roles         → user_roles                             │
│  students + courses    → student_courses                        │
│  tags + articles       → article_tags                           │
│                                                                 │
│  OPTION B: Use a domain-specific name (when the link has        │
│  meaning beyond "connects A and B")                             │
│  ──────────────────────────────────                             │
│  orders + menu_items   → order_items     (not order_menu_items) │
│  users + organizations → memberships     (not user_organizations│
│  users + restaurants   → reviews         (the link IS a review) │
│                                                                 │
│  PREFER Option B when the junction has its own identity.        │
│  "order_items" is a real business concept.                      │
│  "memberships" is a real business concept.                      │
│  Use Option A for pure associations (user_roles, article_tags). │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Enforcing uniqueness on junction tables:**

```sql
-- A customer can only review a restaurant ONCE
-- (from our requirements)
CREATE TABLE reviews (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(id),
    restaurant_id   BIGINT NOT NULL REFERENCES restaurants(id),
    rating          INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- This UNIQUE constraint enforces the business rule
    UNIQUE (user_id, restaurant_id)
);

-- Attempt to insert a second review:
-- INSERT INTO reviews (user_id, restaurant_id, rating) VALUES (1, 1, 4);
-- → ERROR: duplicate key value violates unique constraint
```

---

## 2.4 Self-Referential Relationships

**Sometimes a table relates to ITSELF.**

**From our requirements:**

> "Menu items belong to categories. Categories are per-restaurant."

That's straightforward. But what if categories had subcategories?

```
Food
├── Appetizers
│   ├── Hot Appetizers
│   └── Cold Appetizers
├── Main Course
│   ├── Burgers
│   └── Pasta
└── Drinks
    ├── Hot Drinks
    └── Cold Drinks
```

**A table pointing to itself:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SELF-REFERENTIAL RELATIONSHIP                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────────────┐                                      │
│   │     categories       │                                      │
│   │──────────────────────│                                      │
│   │ id             PK    │                                      │
│   │ restaurant_id  FK ───┼──▶ restaurants                       │
│   │ parent_id      FK ───┼──▶ categories (SELF!)                │
│   │ name                 │                                      │
│   │ sort_order           │                                      │
│   └──────────────────────┘                                      │
│            │       ▲                                            │
│            │       │                                            │
│            └───────┘  Points to itself                          │
│                                                                 │
│                                                                 │
│   parent_id is NULL for top-level categories.                   │
│   parent_id points to another category row for subcategories.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In SQL:**

```sql
CREATE TABLE categories (
    id              BIGSERIAL PRIMARY KEY,
    restaurant_id   BIGINT NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
    parent_id       BIGINT REFERENCES categories(id),  -- NULL = top-level
    name            TEXT NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0
);
```

**The data looks like:**

```
id │ restaurant_id │ parent_id │ name
───┼───────────────┼───────────┼─────────────────
 1 │ 1             │ NULL      │ Food             ← top-level
 2 │ 1             │ 1         │ Appetizers       ← child of Food
 3 │ 1             │ 1         │ Main Course      ← child of Food
 4 │ 1             │ 2         │ Hot Appetizers   ← child of Appetizers
 5 │ 1             │ 2         │ Cold Appetizers  ← child of Appetizers
 6 │ 1             │ NULL      │ Drinks           ← top-level
```

**Querying self-referential trees (Connection to Lecture 2 — CTEs):**

```sql
-- Get a category and all its children (one level)
SELECT child.name AS subcategory
FROM categories AS child
WHERE child.parent_id = 2;  -- Children of "Appetizers"

-- Get the full tree (recursive CTE — Lecture 2 callback)
WITH RECURSIVE category_tree AS (
    -- Anchor: start from top-level
    SELECT id, name, parent_id, 0 AS depth
    FROM categories
    WHERE parent_id IS NULL AND restaurant_id = 1

    UNION ALL

    -- Recursive: find children
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT depth, name FROM category_tree ORDER BY depth, name;
```

```
depth │ name
──────┼─────────────────
    0 │ Drinks
    0 │ Food
    1 │ Appetizers
    1 │ Main Course
    2 │ Cold Appetizers
    2 │ Hot Appetizers
```

**When to use self-referential relationships:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SELF-REFERENTIAL USE CASES                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ GOOD USE CASES:                                             │
│  ├─ Categories / subcategories (menu, product taxonomy)         │
│  ├─ Organizational hierarchies (manager → reports)              │
│  ├─ Comment replies (comment → parent comment)                  │
│  ├─ File/folder structures (folder → parent folder)             │
│  └─ Task dependencies (task → parent task)                      │
│                                                                 │
│  ⚠️  WATCH OUT FOR:                                             │
│  ├─ Deep nesting (recursive queries get expensive)              │
│  ├─ Circular references (A → B → C → A — validate this!)       │
│  └─ Orphaned branches (delete parent without handling children) │
│                                                                 │
│  FOR QUICKBITE:                                                 │
│  We keep it simple — categories are FLAT (no parent_id).        │
│  The requirements say "Appetizers, Main Course, Desserts."      │
│  No subcategories mentioned. Don't over-engineer.               │
│  Add parent_id LATER if the PM asks for nested categories.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to prior knowledge:**

> "Notice we showed the self-referential concept even though QuickBite doesn't need it yet. This is a pattern you'll see in your capstone — project management tools have task hierarchies, team structures, nested comments. You now have the tool. Use it when the requirements demand it, not before."

---

# PART 3: PRIMARY KEY STRATEGIES

## 3.1 Serial IDs: Simple and Sequential

**The default choice most tutorials teach you:**

```sql
CREATE TABLE users (
    id    BIGSERIAL PRIMARY KEY,  -- 1, 2, 3, 4, ...
    email TEXT NOT NULL UNIQUE,
    name  TEXT NOT NULL
);
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  SERIAL / BIGSERIAL                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HOW IT WORKS:                                                  │
│  PostgreSQL auto-increments. First row gets 1, next gets 2.     │
│  SERIAL = INTEGER (max ~2.1 billion)                            │
│  BIGSERIAL = BIGINT (max ~9.2 quintillion)                      │
│                                                                 │
│  ✅ PROS:                                                       │
│  ├─ Small size (4 or 8 bytes)                                   │
│  ├─ Human-readable (User #42 is easier than                     │
│  │  User a3f7b2c1-...)                                          │
│  ├─ Sequential = great index performance                        │
│  │  (B-tree indexes love ordered inserts — Lecture 3 callback)  │
│  ├─ Natural ordering (higher ID = newer record)                 │
│  └─ Simple to debug ("check order 1547")                        │
│                                                                 │
│  ❌ CONS:                                                       │
│  ├─ PREDICTABLE — user can guess other IDs                      │
│  │  GET /api/users/1, GET /api/users/2, GET /api/users/3...     │
│  │  Enumeration attack! (Week 9 — security context)             │
│  ├─ Leaks business info — "your order is #50342"                │
│  │  → competitor knows you've had ~50,000 orders                │
│  ├─ Collision across databases — if you have two databases      │
│  │  both generating IDs, they WILL collide                      │
│  └─ Gaps are confusing (but normal!) — deleted ID 5 means       │
│     IDs go 1, 2, 3, 4, 6 — users ask "where's 5?"             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 UUIDs: Globally Unique

**UUIDs look like this:** `a3f7b2c1-8d4e-4f9a-b5c6-1234567890ab`

```sql
-- PostgreSQL has native UUID support (Lecture 3 callback)
CREATE TABLE users (
    id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    name  TEXT NOT NULL
);
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  UUID                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HOW IT WORKS:                                                  │
│  128-bit random (v4) or time-sortable (v7) identifier.          │
│  gen_random_uuid() generates a v4 UUID in PostgreSQL.           │
│  Probability of collision: effectively zero.                    │
│                                                                 │
│  ✅ PROS:                                                       │
│  ├─ Globally unique — can generate IDs ANYWHERE                 │
│  │  (client, server, different databases) with no coordination  │
│  ├─ Not guessable — no enumeration attacks                      │
│  │  (good for public-facing APIs — Week 9 security context)     │
│  ├─ Doesn't leak volume — order UUID reveals nothing            │
│  ├─ Safe for distributed systems (multiple servers)             │
│  └─ Can be generated before INSERT (useful for related inserts) │
│                                                                 │
│  ❌ CONS:                                                       │
│  ├─ Larger (16 bytes vs 8 for BIGINT)                           │
│  │  → bigger indexes, more memory, slower JOINs                 │
│  ├─ Random UUIDs (v4) fragment indexes                          │
│  │  → inserts scattered across the B-tree, not appended         │
│  │  → slower writes at very high volume                         │
│  ├─ Ugly to humans ("check order a3f7b2c1-8d4e..." — what?)    │
│  └─ Harder to debug ("the bug is in user... hang on let me     │
│     copy-paste this UUID...")                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**UUIDv7 — the modern compromise:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  UUID v4 VS v7                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  UUID v4 (random):                                              │
│  a3f7b2c1-8d4e-4f9a-b5c6-1234567890ab                          │
│       ▲                                                         │
│       └── Completely random. No ordering.                       │
│                                                                 │
│  UUID v7 (time-sortable):                                       │
│  018e4a2c-7f00-7abc-9def-1234567890ab                           │
│  ▲▲▲▲▲▲▲▲                                                      │
│  └── First 48 bits = millisecond timestamp                      │
│      Rest = random                                              │
│                                                                 │
│  v7 gives you:                                                  │
│  ├─ Global uniqueness (like v4)                                 │
│  ├─ Time-sortable (like SERIAL — newer IDs sort after older)    │
│  ├─ Better index performance (sequential-ish inserts)           │
│  └─ Embedded timestamp (can extract creation time from ID)      │
│                                                                 │
│  ⚠️  PostgreSQL's gen_random_uuid() generates v4 only.          │
│  For v7, you'd use an extension or generate in application.     │
│  For this course, v4 is fine. Know that v7 exists.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 The Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│            SERIAL vs UUID: WHEN TO USE WHICH                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                     START HERE                                  │
│                         │                                       │
│                         ▼                                       │
│              ┌─────────────────────┐                            │
│              │ Is this ID exposed  │                            │
│              │ in a public API     │                            │
│              │ or URL?             │                            │
│              └──────────┬──────────┘                            │
│                   │           │                                 │
│                  YES          NO                                │
│                   │           │                                 │
│                   ▼           ▼                                 │
│          ┌──────────────┐  ┌────────────────────┐               │
│          │ Use UUID     │  │ Does the table     │               │
│          │ (prevent     │  │ need IDs generated │               │
│          │  enumeration)│  │ across multiple    │               │
│          └──────────────┘  │ systems?           │               │
│                            └──────────┬─────────┘               │
│                               │           │                    │
│                              YES          NO                   │
│                               │           │                    │
│                               ▼           ▼                    │
│                      ┌──────────────┐  ┌──────────────┐         │
│                      │ Use UUID     │  │ BIGSERIAL is │         │
│                      └──────────────┘  │ fine. Keep   │         │
│                                        │ it simple.   │         │
│                                        └──────────────┘         │
│                                                                 │
│                                                                 │
│  COMMON PATTERN: Use BOTH                                       │
│  ─────────────────────────                                      │
│  Internal PK: BIGSERIAL (fast JOINs, compact indexes)           │
│  External ID: UUID column with unique index (API-facing)        │
│                                                                 │
│  This gives you the best of both worlds, at the cost            │
│  of one extra column and index per table.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The dual-ID pattern:**

```sql
CREATE TABLE orders (
    -- Internal: fast JOINs, compact
    id          BIGSERIAL PRIMARY KEY,

    -- External: exposed in API, no enumeration
    public_id   UUID NOT NULL DEFAULT gen_random_uuid() UNIQUE,

    user_id     BIGINT NOT NULL REFERENCES users(id),
    -- ...
);

-- Internal queries (service-to-service, JOINs):
SELECT * FROM order_items WHERE order_id = 42;

-- API-facing queries (user-facing):
SELECT * FROM orders WHERE public_id = 'a3f7b2c1-8d4e-4f9a-b5c6-1234567890ab';
```

**For QuickBite (our design decision):**

```
┌─────────────────────────────────────────────────────────────────┐
│                  QUICKBITE PK DECISIONS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Table           │ PK Strategy   │ Reasoning                    │
│  ────────────────┼───────────────┼─────────────────────────     │
│  users           │ UUID          │ Exposed in API, security     │
│  restaurants     │ UUID          │ Exposed in API, public URLs  │
│  categories      │ BIGSERIAL     │ Internal only, never in URLs │
│  menu_items      │ UUID          │ Exposed in API (menu links)  │
│  orders          │ UUID          │ Exposed to customers         │
│  order_items     │ BIGSERIAL     │ Internal only, never exposed │
│  reviews         │ UUID          │ Exposed in API (review links)│
│                                                                 │
│  RULE WE CHOSE:                                                 │
│  UUID for anything a user or external system might reference.   │
│  BIGSERIAL for purely internal tables.                          │
│                                                                 │
│  This is ONE valid approach. Not the ONLY one.                  │
│  UUID everywhere is also valid (simpler, slightly slower).      │
│  BIGSERIAL everywhere is also valid (faster, less secure).      │
│  The key is: DECIDE CONSISTENTLY and DOCUMENT WHY.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Composite Keys: When One Column Isn't Enough

**Sometimes a row's identity IS the combination of two foreign keys.**

```sql
-- A user can favorite many restaurants.
-- A restaurant can be favorited by many users.
-- But a user can only favorite a restaurant ONCE.

-- Option A: Surrogate key + unique constraint
CREATE TABLE user_favorites (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    restaurant_id   BIGINT NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
    favorited_at    TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (user_id, restaurant_id)
);

-- Option B: Composite primary key (no surrogate ID)
CREATE TABLE user_favorites (
    user_id         BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    restaurant_id   BIGINT NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
    favorited_at    TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (user_id, restaurant_id)
);
```

**When to use which:**

```
┌─────────────────────────────────────────────────────────────────┐
│           SURROGATE KEY vs COMPOSITE KEY                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SURROGATE KEY (Option A):                                      │
│  ├─ ✅ Easier to reference from other tables (single FK)        │
│  ├─ ✅ ORMs (SQLAlchemy) handle it more naturally               │
│  ├─ ✅ Consistent pattern — every table has an `id`             │
│  └─ ❌ Extra column, extra index, extra storage                 │
│                                                                 │
│  COMPOSITE KEY (Option B):                                      │
│  ├─ ✅ No redundant column — the PK IS the business rule        │
│  ├─ ✅ One fewer index (PK already enforces uniqueness)         │
│  ├─ ✅ Clearly communicates: "this row IS the pair"             │
│  └─ ❌ Awkward if other tables need to reference it             │
│  └─ ❌ Some ORMs make this harder to work with                  │
│                                                                 │
│  IN PRACTICE:                                                   │
│  ├─ Pure junction tables (no other tables reference them)       │
│  │  → composite keys work great                                │
│  ├─ Junction tables that OTHER tables reference                 │
│  │  → surrogate key is easier                                  │
│  └─ When in doubt → surrogate key. Simpler in application code.│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: DATA LIFECYCLE PATTERNS

## 4.1 Hard Deletes: Gone Forever

**The simplest approach: DELETE removes the row permanently.**

```sql
-- Hard delete: row is gone from the table
DELETE FROM menu_items WHERE id = 42;

-- It's as if it never existed.
-- No trace. No recovery (without backups).
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  HARD DELETE                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE DELETE:                                                 │
│  ┌──────────────────────────────────────────┐                   │
│  │ id │ name           │ price │ available  │                   │
│  │  1 │ Classic Burger │ 12.99 │ true       │                   │
│  │  2 │ Fries          │  4.99 │ true       │                   │
│  │  3 │ Cola           │  2.99 │ true       │                   │
│  └──────────────────────────────────────────┘                   │
│                                                                 │
│  DELETE FROM menu_items WHERE id = 2;                           │
│                                                                 │
│  AFTER DELETE:                                                  │
│  ┌──────────────────────────────────────────┐                   │
│  │ id │ name           │ price │ available  │                   │
│  │  1 │ Classic Burger │ 12.99 │ true       │                   │
│  │  3 │ Cola           │  2.99 │ true       │                   │
│  └──────────────────────────────────────────┘                   │
│                                                                 │
│  Row 2 is GONE. No trace in this table.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The cascade problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  HARD DELETE DANGERS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Scenario: Restaurant owner removes "Fries" from the menu.     │
│                                                                 │
│  DELETE FROM menu_items WHERE id = 2;                           │
│                                                                 │
│  But order_items has rows referencing menu_item_id = 2!         │
│                                                                 │
│  If FK is RESTRICT → ERROR. Can't delete.                       │
│  If FK is CASCADE → All order_items with Fries are DELETED.     │
│                     Past orders now have missing line items!     │
│  If FK is SET NULL → order_items.menu_item_id = NULL.           │
│                      "You ordered... something. $4.99."         │
│                                                                 │
│  ALL THREE OPTIONS ARE BAD.                                     │
│  This is a signal: maybe hard delete is wrong here.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Soft Deletes: Hidden but Preserved

**The row stays in the database. A flag marks it as "deleted."**

```sql
-- Soft delete: mark as deleted, don't remove
UPDATE menu_items
SET deleted_at = now()
WHERE id = 2;

-- The row still exists! It's just hidden.
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  SOFT DELETE                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE SOFT DELETE:                                            │
│  ┌──────────────────────────────────────────────────────┐       │
│  │ id │ name           │ price │ available │ deleted_at │       │
│  │  1 │ Classic Burger │ 12.99 │ true      │ NULL       │       │
│  │  2 │ Fries          │  4.99 │ true      │ NULL       │       │
│  │  3 │ Cola           │  2.99 │ true      │ NULL       │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                 │
│  UPDATE menu_items SET deleted_at = now() WHERE id = 2;         │
│                                                                 │
│  AFTER SOFT DELETE:                                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ id │ name           │ price │ available │ deleted_at     │   │
│  │  1 │ Classic Burger │ 12.99 │ true      │ NULL           │   │
│  │  2 │ Fries          │  4.99 │ true      │ 2025-06-15...  │   │
│  │  3 │ Cola           │  2.99 │ true      │ NULL           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Row 2 still exists. Foreign keys still work.                   │
│  Past orders can still reference it.                            │
│  But active queries EXCLUDE it.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Two common implementations:**

```sql
-- Option A: Boolean flag
ALTER TABLE menu_items ADD COLUMN is_deleted BOOLEAN NOT NULL DEFAULT false;

-- Simple but: WHEN was it deleted? You don't know.

-- Option B: Timestamp (PREFERRED)
ALTER TABLE menu_items ADD COLUMN deleted_at TIMESTAMPTZ;

-- deleted_at IS NULL → active
-- deleted_at IS NOT NULL → soft-deleted (and you know WHEN)
```

**The cost: EVERY query must filter:**

```sql
-- ❌ FORGOT THE FILTER — shows deleted items!
SELECT * FROM menu_items WHERE restaurant_id = 1;

-- ✅ CORRECT — exclude soft-deleted
SELECT * FROM menu_items
WHERE restaurant_id = 1
  AND deleted_at IS NULL;

-- This is easy to forget. Every. Single. Query.
-- In Week 6 (SQLAlchemy), you'll learn to automate this
-- with query filters on the ORM model. For now, be disciplined.
```

**Unique constraints with soft deletes get tricky:**

```sql
-- Business rule: menu item names must be unique per restaurant.
-- Simple unique constraint:
UNIQUE (restaurant_id, name)

-- Problem: I soft-delete "Fries", then try to add a NEW "Fries".
-- → ERROR: unique constraint violation!
-- The soft-deleted row still occupies that unique slot.

-- Solution: Partial unique index (Lecture 3 callback)
CREATE UNIQUE INDEX idx_menu_items_unique_name
ON menu_items (restaurant_id, name)
WHERE deleted_at IS NULL;

-- Only ACTIVE rows participate in uniqueness check.
-- Soft-deleted rows are excluded from the index.
```

---

## 4.3 The Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│           HARD DELETE vs SOFT DELETE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                     START HERE                                  │
│                         │                                       │
│                         ▼                                       │
│              ┌─────────────────────┐                            │
│              │ Do other records    │                            │
│              │ reference this one? │                            │
│              │ (FK relationships)  │                            │
│              └──────────┬──────────┘                            │
│                   │           │                                 │
│                  YES          NO                                │
│                   │           │                                 │
│                   ▼           ▼                                 │
│         ┌──────────────┐  ┌──────────────────┐                  │
│         │ Soft delete  │  │ Is there a legal │                  │
│         │ (preserve    │  │ or audit reason  │                  │
│         │ references)  │  │ to keep it?      │                  │
│         └──────────────┘  └──────────┬───────┘                  │
│                               │           │                    │
│                              YES          NO                   │
│                               │           │                    │
│                               ▼           ▼                    │
│                     ┌──────────────┐  ┌──────────────┐          │
│                     │ Soft delete  │  │ Hard delete  │          │
│                     │ (compliance) │  │ is fine      │          │
│                     └──────────────┘  └──────────────┘          │
│                                                                 │
│                                                                 │
│  QUICKBITE DECISIONS:                                           │
│  ─────────────────────                                          │
│  users        → Soft delete (orders reference them)             │
│  restaurants  → Soft delete (orders, reviews reference them)    │
│  categories   → Soft delete (menu_items reference them)         │
│  menu_items   → Soft delete (order_items reference them)        │
│  orders       → Soft delete (financial/legal records)           │
│  order_items  → Hard delete ONLY if parent order is hard-deleted│
│                 (but since orders are soft-deleted, no need)     │
│  reviews      → Hard delete is OK (no children reference them,  │
│                 but soft delete is safer for moderation audits)  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**An honest tradeoff summary:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SOFT DELETE TRADEOFFS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ PROS:                                                       │
│  ├─ Data recovery (undo accidental deletes)                     │
│  ├─ Audit trail (what was deleted, when, by whom)               │
│  ├─ Referential integrity preserved (no orphaned FKs)           │
│  ├─ Legal compliance (data retention requirements)              │
│  └─ Analytics (include deleted records in historical reports)   │
│                                                                 │
│  ❌ CONS:                                                       │
│  ├─ EVERY query needs WHERE deleted_at IS NULL                  │
│  │  (easy to forget → bugs showing "ghost" data)               │
│  ├─ Table grows forever (deleted rows consume storage)          │
│  ├─ Unique constraints are more complex (partial indexes)       │
│  ├─ GDPR/privacy: "delete my data" means actual deletion,      │
│  │  not a flag. Soft delete may not satisfy legal requirements. │
│  └─ Cognitive overhead (is it really deleted? or just hidden?)  │
│                                                                 │
│  PRAGMATIC ADVICE:                                              │
│  Use soft deletes for entities that are REFERENCED by others.   │
│  Use hard deletes for leaf entities with no children.           │
│  For GDPR: soft delete first, then batch-purge after            │
│  retention period with a background job (Week 11 callback).     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Timestamps: Tracking When Things Happen

**Every table needs to answer: "When was this created? When was it last changed?"**

```sql
-- The standard pattern: created_at + updated_at on every table
CREATE TABLE restaurants (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    address     TEXT NOT NULL,
    owner_id    UUID NOT NULL REFERENCES users(id),

    -- Lifecycle timestamps
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ  -- NULL = active (soft delete)
);
```

**TIMESTAMP vs TIMESTAMPTZ (Connection to Lecture 3):**

```
┌─────────────────────────────────────────────────────────────────┐
│              ALWAYS USE TIMESTAMPTZ                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TIMESTAMP (without time zone):                                 │
│  ├─ Stores: 2025-06-15 14:30:00                                │
│  ├─ No timezone info                                            │
│  └─ 2pm WHERE? London? Tokyo? Ambiguous.                       │
│                                                                 │
│  TIMESTAMPTZ (with time zone):                                  │
│  ├─ Stores: 2025-06-15 14:30:00+00 (UTC internally)            │
│  ├─ PostgreSQL converts to UTC on storage                       │
│  ├─ Converts to session timezone on retrieval                   │
│  └─ Unambiguous. Always correct.                                │
│                                                                 │
│  RULE: Always use TIMESTAMPTZ. Never use TIMESTAMP.             │
│  There is no good reason to use TIMESTAMP in a backend app.     │
│  Your users are in different timezones. Your servers might be.  │
│  TIMESTAMPTZ handles all of it correctly.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Handling updated_at automatically:**

```sql
-- Option A: Database trigger (PostgreSQL handles it for you)

-- First, create a reusable function
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Then, attach it to each table
CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON restaurants
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();

-- Now: any UPDATE on restaurants automatically sets updated_at.
-- You CANNOT forget. The database enforces it.
```

```
┌─────────────────────────────────────────────────────────────────┐
│          DATABASE TRIGGER vs APPLICATION CODE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DATABASE TRIGGER:                                              │
│  ├─ ✅ Can't be forgotten (enforced at DB level)                │
│  ├─ ✅ Works even for raw SQL updates                           │
│  ├─ ✅ Works across ALL applications hitting this DB            │
│  ├─ ❌ Hidden logic (developers might not know it exists)       │
│  └─ ❌ Harder to test (need actual DB to verify)               │
│                                                                 │
│  APPLICATION CODE (set updated_at in Python):                   │
│  ├─ ✅ Visible in code (easy to find and understand)            │
│  ├─ ✅ Testable without a database                              │
│  ├─ ❌ Easy to forget in some code paths                        │
│  └─ ❌ Only works through YOUR application                      │
│                                                                 │
│  RECOMMENDATION:                                                │
│  Use BOTH. Set updated_at in application code (explicit).       │
│  Add the trigger as a safety net (in case someone misses it).   │
│  Belt AND suspenders.                                           │
│                                                                 │
│  In Week 6 (SQLAlchemy), you'll see:                            │
│  updated_at = mapped_column(                                    │
│      DateTime(timezone=True),                                   │
│      server_default=func.now(),                                 │
│      onupdate=func.now()  ← SQLAlchemy handles it              │
│  )                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Domain-specific timestamps — go beyond created/updated:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  DOMAIN TIMESTAMPS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Generic timestamps (EVERY table):                              │
│  ├─ created_at    — when the row was inserted                   │
│  ├─ updated_at    — when the row was last modified              │
│  └─ deleted_at    — when the row was soft-deleted (if using)    │
│                                                                 │
│  Domain-specific timestamps (per business logic):               │
│                                                                 │
│  orders:                                                        │
│  ├─ placed_at      — when customer submitted the order          │
│  ├─ confirmed_at   — when restaurant accepted it                │
│  ├─ prepared_at    — when food was ready                        │
│  ├─ picked_up_at   — when driver collected it                   │
│  ├─ delivered_at   — when customer received it                  │
│  └─ cancelled_at   — when/if order was cancelled                │
│                                                                 │
│  These timestamps ENCODE STATE TRANSITIONS.                     │
│  If delivered_at IS NOT NULL → order is delivered.              │
│  If cancelled_at IS NOT NULL → order was cancelled.             │
│  You can derive the current status from timestamps alone.       │
│                                                                 │
│  ⚠️  You may ALSO want an explicit status column               │
│  (status TEXT CHECK (status IN ('placed', 'confirmed', ...)))   │
│  for easy querying. Timestamps and status are complementary:    │
│  status = WHAT state it's in. Timestamps = WHEN it changed.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: THE COMPLETE DESIGN PROCESS

## 5.1 The Step-by-Step Method

**A repeatable process you can apply to ANY domain:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE SCHEMA DESIGN PROCESS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: READ requirements. Underline NOUNS and VERBS.          │
│          │                                                      │
│          ▼                                                      │
│  STEP 2: LIST candidate entities. FILTER to real tables.        │
│          │                                                      │
│          ▼                                                      │
│  STEP 3: For each entity, LIST its attributes (columns).        │
│          Determine data types. Identify NOT NULL vs nullable.   │
│          │                                                      │
│          ▼                                                      │
│  STEP 4: IDENTIFY relationships between entities.               │
│          Determine cardinality: 1:1, 1:N, or M:N.              │
│          │                                                      │
│          ▼                                                      │
│  STEP 5: PLACE foreign keys (on the MANY side for 1:N).         │
│          CREATE junction tables for M:N.                        │
│          │                                                      │
│          ▼                                                      │
│  STEP 6: CHOOSE primary keys (serial vs UUID) per table.        │
│          │                                                      │
│          ▼                                                      │
│  STEP 7: ADD lifecycle columns.                                 │
│          created_at, updated_at on everything.                  │
│          deleted_at where soft deletes are needed.              │
│          Domain-specific timestamps where relevant.             │
│          │                                                      │
│          ▼                                                      │
│  STEP 8: ADD constraints.                                       │
│          NOT NULL, UNIQUE, CHECK, FOREIGN KEY actions.          │
│          │                                                      │
│          ▼                                                      │
│  STEP 9: DRAW the ER diagram. Visual sanity check.              │
│          │                                                      │
│          ▼                                                      │
│  STEP 10: REVIEW against normalization (Lecture 1).              │
│           Is any data duplicated? Can update anomalies happen?  │
│           │                                                     │
│           ▼                                                     │
│  STEP 11: WRITE the SQL. Create tables in dependency order.     │
│           (Tables with no FKs first, then dependent tables.)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 QuickBite: Full Schema Walkthrough

**Let's walk through the complete QuickBite schema, applying everything.**

**The ER diagram:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  QUICKBITE ER DIAGRAM                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  ┌──────────┐    owns     ┌──────────────┐    has     ┌───────┐ │
│  │  users   │────(1:N)───▶│ restaurants  │───(1:N)───▶│ cats  │ │
│  │          │             │              │            │       │ │
│  │  PK: id  │             │  PK: id      │            │PK: id │ │
│  └──────────┘             └──────────────┘            └───┬───┘ │
│       │                        │    │                     │     │
│       │ places                 │    │ receives            │has  │
│       │ (1:N)                  │    │ (1:N)               │(1:N)│
│       │                       │    │                     │     │
│       ▼                       │    ▼                     ▼     │
│  ┌──────────┐         has     │  ┌──────────┐    ┌───────────┐ │
│  │  orders  │────────(1:N)────┘  │ reviews  │    │menu_items │ │
│  │          │                    │          │    │           │ │
│  │  PK: id  │                    │  PK: id  │    │  PK: id   │ │
│  └────┬─────┘                    └──────────┘    └─────┬─────┘ │
│       │ contains                                       │       │
│       │ (1:N)               references (N:1)           │       │
│       │              ┌─────────────────────────────────┘       │
│       ▼              ▼                                         │
│  ┌──────────────────────┐                                      │
│  │    order_items       │                                      │
│  │                      │                                      │
│  │  PK: id              │                                      │
│  │  FK: order_id        │                                      │
│  │  FK: menu_item_id    │                                      │
│  │  quantity, unit_price│                                      │
│  └──────────────────────┘                                      │
│                                                                 │
│  (cats = categories, abbreviated for diagram)                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now the complete SQL, table by table, in dependency order:**

```sql
-- =============================================================
-- QUICKBITE SCHEMA — COMPLETE
-- Tables created in dependency order (no FK references unresolved)
-- =============================================================


-- 1. USERS (no foreign keys — create first)
-- ----------------------------------------------------------
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       TEXT NOT NULL UNIQUE,
    name        TEXT NOT NULL,
    phone       TEXT,                      -- nullable: not required at signup
    role        TEXT NOT NULL DEFAULT 'customer'
                CHECK (role IN ('customer', 'owner', 'admin')),

    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ               -- soft delete
);
```

**Design note on `role`:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHY role IS A TEXT + CHECK                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Three options for storing a fixed set of values:               │
│                                                                 │
│  1. TEXT + CHECK constraint (what we chose)                     │
│     ✅ Simple, readable, easy to query                          │
│     ✅ CHECK enforces valid values at DB level                  │
│     ❌ Adding a new role requires ALTER TABLE                   │
│                                                                 │
│  2. PostgreSQL ENUM type                                        │
│     ✅ Type-safe at DB level                                    │
│     ❌ Adding values requires ALTER TYPE (awkward in migrations)│
│     ❌ Cannot remove values at all                              │
│                                                                 │
│  3. Separate roles table + FK                                   │
│     ✅ Most flexible (add/remove roles anytime)                 │
│     ❌ Requires a JOIN for every user query                     │
│     ❌ Over-engineering for 3 static values                     │
│                                                                 │
│  DECISION: TEXT + CHECK. Simple, good enough for now.           │
│  If roles become complex (permissions per role), migrate        │
│  to a roles table in a future sprint.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```sql
-- 2. RESTAURANTS (depends on users)
-- ----------------------------------------------------------
CREATE TABLE restaurants (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id    UUID NOT NULL REFERENCES users(id),  -- who owns this restaurant
    name        TEXT NOT NULL,
    address     TEXT NOT NULL,
    phone       TEXT NOT NULL,
    description TEXT,

    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ
);


-- 3. CATEGORIES (depends on restaurants)
-- ----------------------------------------------------------
CREATE TABLE categories (
    id              BIGSERIAL PRIMARY KEY,    -- internal only, never in URLs
    restaurant_id   UUID NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()

    -- No deleted_at: if a restaurant deletes a category,
    -- we reassign its menu items first, then hard delete.
    -- Simple. No ghost categories.
);

-- A restaurant can't have two categories with the same name
CREATE UNIQUE INDEX idx_categories_unique_name
ON categories (restaurant_id, name);


-- 4. MENU ITEMS (depends on restaurants, categories)
-- ----------------------------------------------------------
CREATE TABLE menu_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    restaurant_id   UUID NOT NULL REFERENCES restaurants(id),
    category_id     BIGINT REFERENCES categories(id) ON DELETE SET NULL,
                    -- SET NULL: if category deleted, item becomes uncategorized
    name            TEXT NOT NULL,
    description     TEXT,
    price           NUMERIC(10, 2) NOT NULL CHECK (price > 0),
    is_available    BOOLEAN NOT NULL DEFAULT true,  -- can be marked unavailable

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ          -- soft delete (referenced by order_items)
);

-- Menu item names unique per restaurant (only among active items)
CREATE UNIQUE INDEX idx_menu_items_unique_name
ON menu_items (restaurant_id, name)
WHERE deleted_at IS NULL;


-- 5. ORDERS (depends on users, restaurants)
-- ----------------------------------------------------------
CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    restaurant_id   UUID NOT NULL REFERENCES restaurants(id),
    status          TEXT NOT NULL DEFAULT 'placed'
                    CHECK (status IN (
                        'placed', 'confirmed', 'preparing',
                        'ready', 'picked_up', 'delivered', 'cancelled'
                    )),
    total_amount    NUMERIC(10, 2) NOT NULL CHECK (total_amount >= 0),
    delivery_address TEXT NOT NULL,       -- snapshot! not a reference to user address
    notes           TEXT,                 -- "Ring doorbell twice"

    -- Domain timestamps: encode state transitions
    placed_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    confirmed_at    TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,

    -- Generic timestamps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()

    -- No deleted_at: orders are financial records.
    -- Never delete. Use 'cancelled' status instead.
);


-- 6. ORDER ITEMS (depends on orders, menu_items)
-- ----------------------------------------------------------
CREATE TABLE order_items (
    id              BIGSERIAL PRIMARY KEY,    -- internal only
    order_id        UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    menu_item_id    UUID NOT NULL REFERENCES menu_items(id),
                    -- No CASCADE: menu item soft-deleted, but order_item stays
    quantity        INTEGER NOT NULL CHECK (quantity > 0),
    unit_price      NUMERIC(10, 2) NOT NULL,  -- SNAPSHOT of price at order time
    item_name       TEXT NOT NULL,             -- SNAPSHOT of name at order time

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
    -- No updated_at: order items don't change after creation.
    -- No deleted_at: lifecycle tied to parent order.
);


-- 7. REVIEWS (depends on users, restaurants)
-- ----------------------------------------------------------
CREATE TABLE reviews (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    restaurant_id   UUID NOT NULL REFERENCES restaurants(id),
    rating          INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment         TEXT,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Business rule: one review per user per restaurant
    UNIQUE (user_id, restaurant_id)
);
```

**Notice the decisions embedded in this schema:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DECISIONS DOCUMENTED INLINE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Every non-obvious choice has a comment explaining WHY.         │
│                                                                 │
│  • ON DELETE CASCADE on order_items — meaningless without order  │
│  • ON DELETE SET NULL on category_id — item survives without it │
│  • unit_price on order_items — snapshot, not live lookup         │
│  • No deleted_at on orders — financial records, never delete    │
│  • No updated_at on order_items — immutable after creation      │
│  • Partial unique index on menu_items — respects soft deletes   │
│                                                                 │
│  YOUR SCHEMA IS DOCUMENTATION.                                  │
│  Future developers (including future you) will read it.         │
│  Comments on constraints explain your INTENT.                   │
│  The SQL alone tells you WHAT. Comments tell you WHY.           │
│                                                                 │
│  Connection to Week 13 (capstone):                              │
│  This practice of documenting design decisions becomes formal   │
│  in Architecture Decision Records (ADRs).                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Common Design Mistakes

### Mistake 1: The God Table (you saw this already)

```sql
-- ❌ Everything in one table
CREATE TABLE everything (
    order_id INT, customer_name TEXT, customer_email TEXT,
    restaurant_name TEXT, item_name TEXT, item_price NUMERIC, ...
);

-- ✅ Separate entities with relationships
-- (the entire schema we just built)
```

> "If your table has more than ~8-10 columns and some of them repeat across rows, you probably need to split."

---

### Mistake 2: Missing Foreign Key Constraints

```sql
-- ❌ "I'll just use matching IDs by convention"
CREATE TABLE order_items (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT,        -- no REFERENCES!
    menu_item_id BIGINT        -- no REFERENCES!
);
-- Nothing prevents inserting order_id = 999999 (nonexistent).
-- Nothing prevents deleting an order while items still point to it.
-- Your data WILL become inconsistent. Not "might." WILL.

-- ✅ Enforce with REFERENCES
CREATE TABLE order_items (
    id          BIGSERIAL PRIMARY KEY,
    order_id    UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    menu_item_id UUID NOT NULL REFERENCES menu_items(id)
);
-- The database guards your integrity. Trust the database.
```

---

### Mistake 3: Wrong Cardinality

```sql
-- Requirement: "An order can contain multiple items."
-- Developer thinks: "I'll put item info on the order."

-- ❌ Cramming many items into one order row
CREATE TABLE orders (
    id          UUID PRIMARY KEY,
    user_id     UUID REFERENCES users(id),
    item1_name  TEXT,       -- What if they order 20 items?
    item1_qty   INTEGER,    -- Do you add 20 pairs of columns?
    item2_name  TEXT,       -- This is a 1:1 disguised as 1:N.
    item2_qty   INTEGER     -- Normalization violation (1NF).
);

-- ✅ Separate table for the "many" side
-- orders + order_items (as we designed)
```

---

### Mistake 4: Nullable Everything

```sql
-- ❌ "Let's make everything nullable just in case"
CREATE TABLE users (
    id      UUID PRIMARY KEY,
    email   TEXT,        -- Can a user have no email?
    name    TEXT,        -- Can a user have no name?
    phone   TEXT,        -- OK this one can be null
    role    TEXT         -- Can a user have no role?
);

-- ✅ Be deliberate about nullability
CREATE TABLE users (
    id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email   TEXT NOT NULL UNIQUE,   -- Required. Must be unique.
    name    TEXT NOT NULL,           -- Required.
    phone   TEXT,                    -- Optional. NULL means "not provided."
    role    TEXT NOT NULL DEFAULT 'customer'  -- Required. Has sensible default.
);
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE NULL RULE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NULL means "UNKNOWN" or "NOT APPLICABLE."                      │
│  It does NOT mean "empty string" or "zero" or "false."          │
│                                                                 │
│  For every column, ask:                                         │
│  "Can this value legitimately be unknown or absent?"            │
│                                                                 │
│  phone = NULL → "User hasn't provided a phone number." ✅       │
│  email = NULL → "User has no email." Can they log in? ❌        │
│  price = NULL → "This item has no price." That's a bug. ❌      │
│                                                                 │
│  DEFAULT to NOT NULL. Make a column nullable only               │
│  when NULL has a specific, meaningful interpretation.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Mistake 5: Storing Computed Values Without Reason

```sql
-- ❌ Storing what you can calculate
CREATE TABLE restaurants (
    id              UUID PRIMARY KEY,
    name            TEXT NOT NULL,
    average_rating  NUMERIC(3, 2),    -- Computed from reviews!
    total_orders    INTEGER           -- Computed from orders!
);
-- These go stale IMMEDIATELY.
-- New review posted → average_rating is wrong until you update it.
-- Dual-write problem: update reviews AND restaurants on every review.

-- ✅ Compute on read (or cache — Week 10)
SELECT
    r.id,
    r.name,
    AVG(rv.rating)::NUMERIC(3,2) AS average_rating,
    COUNT(DISTINCT o.id) AS total_orders
FROM restaurants r
LEFT JOIN reviews rv ON rv.restaurant_id = r.id
LEFT JOIN orders o ON o.restaurant_id = r.id
GROUP BY r.id;

-- If this query is too slow: add a materialized view or cache.
-- That's a performance decision (Week 7, 10), not a schema decision.
```

---

## 5.4 The Design Review Checklist

```
┌─────────────────────────────────────────────────────────────────┐
│              SCHEMA REVIEW CHECKLIST                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Before writing CREATE TABLE, verify:                           │
│                                                                 │
│  ENTITIES                                                       │
│  □ Every noun in requirements maps to a table or column         │
│  □ No entity is missing (re-read requirements!)                 │
│  □ No entity is redundant (would two tables serve one purpose?) │
│                                                                 │
│  RELATIONSHIPS                                                  │
│  □ Every relationship has the correct cardinality (1:1, 1:N, MN)│
│  □ Foreign keys are on the MANY side of 1:N relationships       │
│  □ M:N relationships have junction tables                       │
│  □ ON DELETE behavior is specified for every FK                  │
│  □ No circular dependencies (A→B→C→A)                           │
│                                                                 │
│  COLUMNS                                                        │
│  □ Every NOT NULL has a reason (is this truly required?)        │
│  □ Every nullable column has a reason (what does NULL mean?)    │
│  □ Data types match the data (NUMERIC for money, not FLOAT!)   │
│  □ CHECK constraints enforce business rules at DB level         │
│  □ UNIQUE constraints prevent invalid duplicates                │
│                                                                 │
│  PRIMARY KEYS                                                   │
│  □ Every table has a PK                                         │
│  □ UUID vs SERIAL choice is deliberate and consistent           │
│  □ No business data as PK (emails change, names change)        │
│                                                                 │
│  LIFECYCLE                                                      │
│  □ created_at on every table                                    │
│  □ updated_at on every table (except immutable tables)          │
│  □ Soft delete (deleted_at) where needed                        │
│  □ Domain-specific timestamps where state transitions exist     │
│                                                                 │
│  NORMALIZATION                                                  │
│  □ No data is duplicated across rows unnecessarily              │
│  □ No update anomalies possible                                 │
│  □ Any denormalization is deliberate and documented             │
│                                                                 │
│  DOCUMENTATION                                                  │
│  □ ER diagram exists and matches the SQL                        │
│  □ Non-obvious design choices have comments                     │
│  □ Table and column names are consistent (plural? snake_case?)  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              DATABASE DESIGN QUICK REFERENCE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EXTRACT ENTITIES:                                              │
│      Underline nouns → filter → tables                          │
│      Underline verbs → relationships                            │
│                                                                 │
│  RELATIONSHIPS:                                                 │
│      1:N  → FK on the MANY side                                 │
│      M:N  → Junction table with two FKs                         │
│      Self → FK pointing to same table (parent_id)               │
│                                                                 │
│  PRIMARY KEYS:                                                  │
│      Public-facing → UUID                                       │
│      Internal-only → BIGSERIAL                                  │
│      When in doubt → UUID (safer default)                       │
│                                                                 │
│  ON DELETE:                                                     │
│      Children meaningless without parent → CASCADE              │
│      Children survive independently     → SET NULL              │
│      Prevent accidental data loss       → RESTRICT (default)   │
│                                                                 │
│  SOFT DELETE:                                                   │
│      deleted_at TIMESTAMPTZ (NULL = active)                     │
│      Partial unique indexes: WHERE deleted_at IS NULL           │
│      Remember: every query must filter!                         │
│                                                                 │
│  TIMESTAMPS:                                                    │
│      Always TIMESTAMPTZ, never TIMESTAMP                        │
│      created_at + updated_at on every table                     │
│      Domain timestamps for state transitions                    │
│      Auto-update via trigger or ORM                             │
│                                                                 │
│  SNAPSHOT DATA:                                                 │
│      If source value can change, and you need historical        │
│      accuracy, copy the value at event time.                    │
│      (unit_price on order_items, delivery_address on orders)    │
│                                                                 │
│  NAMING:                                                        │
│      Tables: plural, snake_case (users, order_items)            │
│      Columns: singular, snake_case (user_id, created_at)       │
│      FKs: referenced_table_singular_id (restaurant_id)         │
│      Junction: combined names or domain name                    │
│         (user_roles or memberships)                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  DATABASE DESIGN = ARCHITECTURE                                 │
│                                                                 │
│  You are the architect. The requirements are your client's      │
│  wish list. Your schema is the blueprint. SQL is the            │
│  construction crew. A change order after the building is up     │
│  costs 100x more than erasing a line on the blueprint.          │
│                                                                 │
│                                                                 │
│  THE PROCESS:                                                   │
│                                                                 │
│  Requirements ──▶ Nouns ──▶ Entities ──▶ Relationships          │
│       │                                      │                  │
│       │            ┌─────────────────────────┘                  │
│       │            ▼                                            │
│       │        ER Diagram ──▶ SQL ──▶ Review                    │
│       │                               │                         │
│       └───────────────────────────────┘                         │
│                    iterate                                       │
│                                                                 │
│                                                                 │
│  THE BLUEPRINT ANALOGY:                                         │
│  ├─ Tables = Rooms (each with a clear purpose)                  │
│  ├─ Foreign keys = Doors connecting rooms                       │
│  ├─ Constraints = Building codes (NOT NULL, CHECK, UNIQUE)      │
│  ├─ Junction tables = Shared hallways between wings             │
│  ├─ Primary keys = Room numbers (unique addresses)              │
│  ├─ Soft deletes = Storage room (not the dumpster)              │
│  └─ Timestamps = Security cameras (when did this happen?)       │
│                                                                 │
│  EVERY CHOICE IS A TRADEOFF. Document it.                       │
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
│  WEEK 5 MINI-PROJECT (this week):                               │
│  └─ Design and populate a database from scratch.                │
│     Apply EVERYTHING from this lecture: entities, relationships, │
│     PK strategy, timestamps, soft deletes. ER diagram required. │
│                                                                 │
│  WEEK 6 (SQLAlchemy & Alembic):                                 │
│  └─ Your schema becomes Python classes.                         │
│     CREATE TABLE → class User(Base): ...                        │
│     Every design decision here maps to ORM configuration.       │
│     Alembic migrations evolve the schema you designed.          │
│                                                                 │
│  WEEK 7 (Advanced Database Patterns):                           │
│  └─ Indexes and query optimization for the schema you built.    │
│     EXPLAIN ANALYZE on YOUR tables, YOUR queries.               │
│     Connection pooling, bulk operations, performance tuning.    │
│                                                                 │
│  WEEK 9 (Authentication):                                       │
│  └─ The users table gets password_hash, token storage.          │
│     Role-based access ties directly to the role column.         │
│     UUID PKs prevent the enumeration attacks we warned about.   │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ You'll design a multi-tenant SaaS schema from scratch.      │
│     Organizations, projects, tasks, memberships, permissions.   │
│     The process you learned today IS the process you'll use.    │
│     Architecture Decision Records formalize the "WHY" comments  │
│     we embedded in our SQL today.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```