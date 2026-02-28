# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM FIRST, SYNTAX LAST                                     │
│  ───────────────────────────                                    │
│  Students will SEE the pain of querying data with Python,       │
│  then discover SQL solves it in one readable sentence.          │
│                                                                 │
│  PARADIGM SHIFT: IMPERATIVE → DECLARATIVE                       │
│  ─────────────────────────────────────────                      │
│  The hardest thing about SQL isn't syntax — it's learning       │
│  to describe WHAT you want, not HOW to get it.                  │
│  Every section hammers this shift.                              │
│                                                                 │
│  VISUAL TABLES EVERYWHERE                                       │
│  ────────────────────────                                       │
│  Every query is shown as: Input Table → Query → Output Table.   │
│  Students SEE data transforming, not just read syntax.          │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Lecture 1 (ER diagrams, PKs, FKs) → now we BUILD those        │
│  Week 3-4 (Task Manager API) → we query the same domain        │
│  Week 4 (pagination) → LIMIT/OFFSET is the SQL behind it       │
│  Type hints → SQL has types too (Lecture 1 data types)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                     SQL FUNDAMENTALS                            │
│                     (3.5–4 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (40 min)                                   │
│  ├─ 1.1 The Python Way (Demonstration)                          │
│  ├─ 1.2 The SQL Way (The Reveal)                                │
│  ├─ 1.3 Imperative vs Declarative Thinking                      │
│  ├─ 1.4 Running PostgreSQL with Docker                          │
│  └─ 1.5 Your First psql Session                                 │
│                                                                 │
│  PART 2: THE PLAYGROUND (15 min)                                │
│  ├─ 2.1 CREATE TABLE — Translating Your ER Diagram              │
│  ├─ 2.2 INSERT — Populating Sample Data                         │
│  └─ 2.3 Our Dataset (Visual Overview)                           │
│                                                                 │
│  PART 3: READING DATA — SELECT (50 min)                         │
│  ├─ 3.1 SELECT — Your First Query                               │
│  ├─ 3.2 WHERE — Filtering Rows                                  │
│  ├─ 3.3 Operators and Conditions                                │
│  ├─ 3.4 The NULL Trap                                           │
│  ├─ 3.5 ORDER BY — Sorting Results                              │
│  ├─ 3.6 LIMIT and OFFSET — Pagination                           │
│  └─ 3.7 SQL Execution Order (The Hidden Truth)                  │
│                                                                 │
│  PART 4: MODIFYING DATA — INSERT, UPDATE, DELETE (25 min)       │
│  ├─ 4.1 INSERT — Adding New Rows                                │
│  ├─ 4.2 UPDATE — Changing Existing Rows                         │
│  ├─ 4.3 DELETE — Removing Rows                                  │
│  └─ 4.4 The RETURNING Clause (PostgreSQL Bonus)                 │
│                                                                 │
│  PART 5: JOINS — CONNECTING TABLES (50 min)                     │
│  ├─ 5.1 The Problem (Isolated Tables)                           │
│  ├─ 5.2 INNER JOIN — Only Matching Rows                         │
│  ├─ 5.3 LEFT JOIN — Keep Everything from the Left               │
│  ├─ 5.4 RIGHT JOIN and FULL JOIN                                │
│  ├─ 5.5 Joining Three or More Tables                            │
│  └─ 5.6 Common Join Mistakes                                    │
│                                                                 │
│  PART 6: AGGREGATION — SUMMARIZING DATA (30 min)                │
│  ├─ 6.1 Aggregate Functions (COUNT, SUM, AVG, MIN, MAX)         │
│  ├─ 6.2 GROUP BY — Aggregate per Category                       │
│  ├─ 6.3 HAVING — Filtering Groups                               │
│  └─ 6.4 Aggregation with Joins (Real-World Queries)             │
│                                                                 │
│  PART 7: SUBQUERIES AND CTEs (25 min)                           │
│  ├─ 7.1 Subqueries — Queries Inside Queries                     │
│  ├─ 7.2 CTEs — The WITH Clause                                  │
│  └─ 7.3 When to Use Each                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Python Way

**Start with code they recognize. Make them feel the pain.**

```python
# Remember your Task Manager API from Weeks 3-4?
# All data lived in a Python list:
tasks = [
    {"id": 1, "title": "Fix login bug",  "assignee": "Alice", "status": "done",     "priority": "high"},
    {"id": 2, "title": "Add pagination",  "assignee": "Bob",   "status": "active",   "priority": "medium"},
    {"id": 3, "title": "Write tests",     "assignee": "Alice", "status": "active",   "priority": "high"},
    {"id": 4, "title": "Deploy v2",       "assignee": "Carol", "status": "planning", "priority": "low"},
    {"id": 5, "title": "Update docs",     "assignee": "Bob",   "status": "active",   "priority": "medium"},
    {"id": 6, "title": "Code review",     "assignee": "Alice", "status": "active",   "priority": "high"},
    {"id": 7, "title": "Setup CI",        "assignee": "David", "status": "done",     "priority": "high"},
    {"id": 8, "title": "Refactor auth",   "assignee": "Alice", "status": "planning", "priority": "medium"},
]
```

**Now ask the class some questions. Make them answer in Python.**

> "Question 1: Find all active, high-priority tasks assigned to Alice, sorted by ID."

```python
result = []
for task in tasks:
    if (task["assignee"] == "Alice"
            and task["status"] == "active"
            and task["priority"] == "high"):
        result.append(task)
result.sort(key=lambda t: t["id"])
for task in result:
    print(f"{task['id']}: {task['title']}")
```

> "Question 2: How many tasks does each person have?"

```python
from collections import Counter
counts = Counter(task["assignee"] for task in tasks)
for person, count in sorted(counts.items()):
    print(f"{person}: {count}")
```

> "Question 3: Who has more than 2 tasks?"

```python
busy = {
    name: count
    for name, count in counts.items()
    if count > 2
}
for person, count in busy.items():
    print(f"{person}: {count}")
```

**Now ask the class one more question:**

> "Question 4: What happens when you restart the server?"

Answer: **Everything is gone.** Every task, every category, every user. Gone. That list exists only in memory. The moment the process dies, the data dies with it.

```
┌─────────────────────────────────────────────────────────────────┐
│              THE PROBLEM WITH IN-MEMORY DATA                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. QUERYING IS VERBOSE                                         │
│     └─ 5 lines of Python to answer a simple question            │
│                                                                 │
│  2. QUERYING IS FRAGILE                                         │
│     └─ Typo in "assignee"? No error. Silent wrong results.      │
│                                                                 │
│  3. DATA IS TEMPORARY                                           │
│     └─ Server restart = data gone forever                       │
│                                                                 │
│  4. NO INTEGRITY GUARANTEES                                     │
│     └─ Nothing stops you from inserting {"id": "banana"}        │
│                                                                 │
│  5. DOES NOT SCALE                                              │
│     └─ 1 million tasks? Good luck looping through that list.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.2 The SQL Way

**Now show them the same questions answered with SQL.**

```sql
-- Question 1: Active, high-priority tasks for Alice, sorted by ID
SELECT id, title
FROM tasks
WHERE assignee = 'Alice'
  AND status = 'active'
  AND priority = 'high'
ORDER BY id;
```

```sql
-- Question 2: How many tasks does each person have?
SELECT assignee, COUNT(*) AS task_count
FROM tasks
GROUP BY assignee
ORDER BY assignee;
```

```sql
-- Question 3: Who has more than 2 tasks?
SELECT assignee, COUNT(*) AS task_count
FROM tasks
GROUP BY assignee
HAVING COUNT(*) > 2;
```

**Let the class sit with that comparison for a moment.**

> "Read the SQL out loud. 'SELECT id and title FROM tasks WHERE assignee equals Alice AND status equals active AND priority equals high, ORDER BY id.' It reads like English. You're describing WHAT you want. The database figures out HOW to get it."

> "And when you restart your server? The data is still there. That's what a database is for."

---

## 1.3 Imperative vs Declarative Thinking

**This is the fundamental paradigm shift of this lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│             IMPERATIVE vs DECLARATIVE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IMPERATIVE (Python)                                            │
│  ───────────────────                                            │
│  You tell the computer HOW to do something.                     │
│  Step by step. Loop here. Check this. Append that.              │
│                                                                 │
│     "Go to the shelf. Pick up each book one at a time.          │
│      Open it. Check if the author is 'Tolkien.'                 │
│      If yes, put it in this pile. Move to the next book.        │
│      When done, sort the pile by title."                        │
│                                                                 │
│                                                                 │
│  DECLARATIVE (SQL)                                              │
│  ─────────────────                                              │
│  You tell the computer WHAT you want.                           │
│  Describe the result. The engine figures out the steps.         │
│                                                                 │
│     "Give me all books by Tolkien, sorted by title."            │
│                                                                 │
│                                                                 │
│  SQL is DECLARATIVE.                                            │
│  You describe the result you want.                              │
│  PostgreSQL decides the fastest way to get it.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map it to what they know:**

```
Python (Imperative)            │  SQL (Declarative)
───────────────────────────────│──────────────────────────
for task in tasks:             │  SELECT ... FROM tasks
    if task["status"]=="active"│  WHERE status = 'active'
result.sort(key=...)           │  ORDER BY ...
result[:10]                    │  LIMIT 10
Counter(...)                   │  GROUP BY ... COUNT(*)
```

> "For the rest of this lecture, practice this mental shift. Don't think 'how do I loop through the data?' Think 'what does the result look like?' Describe the result, and SQL will build it."

---

## 1.4 Running PostgreSQL with Docker

**You need a real database. Docker gets you there in one command.**

> "Docker lets you run applications in isolated packages called containers. Think of it as a lightweight, disposable box that holds an entire application and all its dependencies. We'll understand Docker deeply in Week 15. For now, it's a tool that installs and runs PostgreSQL without touching your system."

**Step 1: Create a `docker-compose.yml` file:**

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: student
      POSTGRES_PASSWORD: password
      POSTGRES_DB: workshop
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

**What each line means (briefly):**

```
┌─────────────────────────────────────────────────────────────────┐
│                 docker-compose.yml BREAKDOWN                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  image: postgres:16                                             │
│  └─ "Use the official PostgreSQL version 16 image"              │
│                                                                 │
│  environment:                                                   │
│  ├─ POSTGRES_USER      → The username you'll log in with        │
│  ├─ POSTGRES_PASSWORD   → The password (for dev only!)          │
│  └─ POSTGRES_DB         → The database to create automatically  │
│                                                                 │
│  ports: "5432:5432"                                             │
│  └─ "Map port 5432 inside the container to 5432 on my machine" │
│                                                                 │
│  volumes: pgdata                                                │
│  └─ "Keep the data even if the container restarts"              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Step 2: Start PostgreSQL:**

```bash
docker compose up -d
```

The `-d` flag means "detached" — it runs in the background. You should see:

```
[+] Running 1/1
 ✔ Container workshop-db-1  Started
```

**Step 3: Verify it's running:**

```bash
docker compose ps
```

```
NAME              STATUS          PORTS
workshop-db-1    Up 10 seconds    0.0.0.0:5432->5432/tcp
```

**When you're done for the day:**

```bash
docker compose down       # Stop the container (data persists in volume)
docker compose down -v    # Stop AND delete data (fresh start)
```

---

## 1.5 Your First psql Session

**`psql` is the command-line client for PostgreSQL. It's where you'll write SQL.**

**Connect to your database:**

```bash
docker compose exec db psql -U student -d workshop
```

You should see:

```
psql (16.x)
Type "help" for help.

workshop=>
```

**That `workshop=>` is your SQL prompt. Like Python's `>>>` but for SQL.**

**Essential psql commands (these are NOT SQL — they're psql shortcuts):**

```
┌─────────────────────────────────────────────────────────────────┐
│                   ESSENTIAL psql COMMANDS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  \l          List all databases                                 │
│  \c dbname   Connect to a different database                    │
│  \dt         List all tables in current database                │
│  \d table    Describe a table (columns, types, constraints)     │
│  \x          Toggle expanded display (for wide tables)          │
│  \q          Quit psql                                          │
│                                                                 │
│  NOTE: These start with backslash and have NO semicolon.        │
│        SQL statements MUST end with a semicolon.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The semicolon rule:**

```sql
-- ❌ WRONG: No semicolon — psql waits for more input
SELECT * FROM employees
workshop->              -- Notice: -> means "I'm still waiting"

-- ✅ CORRECT: Semicolon ends the statement
SELECT * FROM employees;
```

> "If your psql prompt changes from `=>` to `->`, it's waiting for you to finish the statement. Type a semicolon and press Enter. If you're really stuck, type `\q` and reconnect."

**Try it now:**

```sql
-- Your first SQL statement!
SELECT 'Hello, SQL!' AS greeting;
```

```
  greeting
─────────────
 Hello, SQL!
(1 row)
```

> "Congratulations. You just asked a database a question, and it answered."

---

# PART 2: THE PLAYGROUND

## 2.1 CREATE TABLE — Translating Your ER Diagram

**In Lecture 1, you designed ER diagrams and learned about tables, columns, primary keys, and foreign keys. Now we translate that into actual SQL.**

> "CREATE TABLE is how you turn an ER diagram into a real database structure. Every concept from Lecture 1 — primary keys, foreign keys, data types, constraints — has a direct SQL equivalent."

```sql
-- Departments: a simple reference table
CREATE TABLE departments (
    id    SERIAL PRIMARY KEY,
    name  TEXT NOT NULL UNIQUE,
    budget INTEGER NOT NULL
);
```

**Connect each line to Lecture 1:**

```
┌─────────────────────────────────────────────────────────────────┐
│           CREATE TABLE ↔ LECTURE 1 CONCEPTS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SQL                          │  Lecture 1 Concept              │
│  ─────────────────────────────│──────────────────────────       │
│  CREATE TABLE departments     │  Entity from ER diagram         │
│  id SERIAL PRIMARY KEY        │  Primary key (auto-increment)   │
│  name TEXT NOT NULL UNIQUE     │  Column with constraints        │
│  budget INTEGER NOT NULL      │  Data type from Lecture 1       │
│  REFERENCES departments(id)  │  Foreign key (relationship)     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now the employees table — this has a foreign key:**

```sql
CREATE TABLE employees (
    id             SERIAL PRIMARY KEY,
    name           TEXT NOT NULL,
    email          TEXT NOT NULL UNIQUE,
    department_id  INTEGER REFERENCES departments(id),
    salary         INTEGER NOT NULL,
    hire_date      DATE NOT NULL
);
```

> "Notice `department_id INTEGER REFERENCES departments(id)`. That's a foreign key. It means every value in this column MUST exist in the `departments.id` column — or be NULL. The database enforces this. Try inserting `department_id = 999` when no department 999 exists — PostgreSQL will reject it."

> "Also notice: `department_id` has no `NOT NULL`. It's NULLABLE by default. An employee might not be assigned to a department yet. This is a deliberate design choice."

**The projects and assignments tables:**

```sql
CREATE TABLE projects (
    id       SERIAL PRIMARY KEY,
    name     TEXT NOT NULL,
    status   TEXT NOT NULL DEFAULT 'planning',
    deadline DATE,
    budget   INTEGER
);

-- Junction table for many-to-many (Lecture 1: many-to-many relationships)
CREATE TABLE assignments (
    employee_id INTEGER REFERENCES employees(id),
    project_id  INTEGER REFERENCES projects(id),
    role        TEXT NOT NULL,
    PRIMARY KEY (employee_id, project_id)
);
```

> "Remember junction tables from Lecture 1? One employee can work on many projects. One project can have many employees. The `assignments` table sits in the middle and holds the relationship — plus the `role` each employee plays in each project. The composite primary key `(employee_id, project_id)` ensures no duplicate assignments."

---

## 2.2 INSERT — Populating Sample Data

**Let's fill our tables with data. You'll learn INSERT formally in Part 4 — for now, just run these statements to build our playground.**

```sql
INSERT INTO departments (name, budget) VALUES
    ('Engineering', 500000),
    ('Marketing',   200000),
    ('Design',      150000),
    ('Sales',       300000);

INSERT INTO employees (name, email, department_id, salary, hire_date) VALUES
    ('Alice Chen',   'alice@company.com',  1, 95000,  '2022-01-15'),
    ('Bob Smith',    'bob@company.com',    1, 85000,  '2022-06-01'),
    ('Carol Jones',  'carol@company.com',  2, 75000,  '2023-03-10'),
    ('David Lee',    'david@company.com',  1, 110000, '2021-09-01'),
    ('Eve Wilson',   'eve@company.com',    3, 80000,  '2023-01-20'),
    ('Frank Brown',  'frank@company.com',  2, 70000,  '2023-07-15'),
    ('Grace Kim',    'grace@company.com',  NULL, 90000, '2024-01-10');

INSERT INTO projects (name, status, deadline, budget) VALUES
    ('API Redesign',    'active',    '2024-06-01', 50000),
    ('Mobile App',      'active',    '2024-08-15', 120000),
    ('Marketing Site',  'completed', '2024-03-01', 30000),
    ('Data Pipeline',   'planning',  '2024-12-01', 80000),
    ('Brand Refresh',   'active',    '2024-07-01', 25000);

INSERT INTO assignments (employee_id, project_id, role) VALUES
    (1, 1, 'lead'),        -- Alice leads API Redesign
    (2, 1, 'developer'),   -- Bob develops on API Redesign
    (4, 1, 'reviewer'),    -- David reviews API Redesign
    (1, 2, 'developer'),   -- Alice develops on Mobile App
    (2, 2, 'lead'),        -- Bob leads Mobile App
    (5, 2, 'designer'),    -- Eve designs Mobile App
    (3, 3, 'lead'),        -- Carol leads Marketing Site
    (6, 3, 'contributor'), -- Frank contributes to Marketing Site
    (5, 3, 'designer'),    -- Eve designs Marketing Site
    (4, 4, 'lead'),        -- David leads Data Pipeline
    (1, 4, 'developer');   -- Alice develops on Data Pipeline
```

**Verify the data loaded:**

```sql
SELECT COUNT(*) FROM employees;
```

```
 count
───────
     7
(1 row)
```

---

## 2.3 Our Dataset

**Before we query, know what we're working with. These are the tables and data you'll query for the rest of this lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                       OUR DATABASE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  departments (4 rows)                                           │
│  ┌────┬─────────────┬────────┐                                  │
│  │ id │    name     │ budget │                                  │
│  ├────┼─────────────┼────────┤                                  │
│  │  1 │ Engineering │ 500000 │                                  │
│  │  2 │ Marketing   │ 200000 │                                  │
│  │  3 │ Design      │ 150000 │                                  │
│  │  4 │ Sales       │ 300000 │                                  │
│  └────┴─────────────┴────────┘                                  │
│                                                                 │
│  employees (7 rows)                                             │
│  ┌────┬──────────────┬───────────────┬────────┬────────────┐    │
│  │ id │    name      │ department_id │ salary │ hire_date  │    │
│  ├────┼──────────────┼───────────────┼────────┼────────────┤    │
│  │  1 │ Alice Chen   │       1       │  95000 │ 2022-01-15 │    │
│  │  2 │ Bob Smith    │       1       │  85000 │ 2022-06-01 │    │
│  │  3 │ Carol Jones  │       2       │  75000 │ 2023-03-10 │    │
│  │  4 │ David Lee    │       1       │ 110000 │ 2021-09-01 │    │
│  │  5 │ Eve Wilson   │       3       │  80000 │ 2023-01-20 │    │
│  │  6 │ Frank Brown  │       2       │  70000 │ 2023-07-15 │    │
│  │  7 │ Grace Kim    │     NULL      │  90000 │ 2024-01-10 │    │
│  └────┴──────────────┴───────────────┴────────┴────────────┘    │
│                                                                 │
│  ⚠️  Notice: Grace has NULL department_id (unassigned).          │
│      This will matter when we learn JOINs.                      │
│                                                                 │
│  projects (5 rows)                                              │
│  ┌────┬────────────────┬───────────┬────────────┬────────┐      │
│  │ id │     name       │  status   │  deadline  │ budget │      │
│  ├────┼────────────────┼───────────┼────────────┼────────┤      │
│  │  1 │ API Redesign   │ active    │ 2024-06-01 │  50000 │      │
│  │  2 │ Mobile App     │ active    │ 2024-08-15 │ 120000 │      │
│  │  3 │ Marketing Site │ completed │ 2024-03-01 │  30000 │      │
│  │  4 │ Data Pipeline  │ planning  │ 2024-12-01 │  80000 │      │
│  │  5 │ Brand Refresh  │ active    │ 2024-07-01 │  25000 │      │
│  └────┴────────────────┴───────────┴────────────┴────────┘      │
│                                                                 │
│  ⚠️  Notice: Brand Refresh has NO assignments.                   │
│      No one is working on it yet. This matters for JOINs too.   │
│                                                                 │
│  assignments (11 rows) — junction table                         │
│  ┌─────────────┬────────────┬─────────────┐                     │
│  │ employee_id │ project_id │    role     │                     │
│  ├─────────────┼────────────┼─────────────┤                     │
│  │      1      │     1      │ lead        │                     │
│  │      2      │     1      │ developer   │                     │
│  │      4      │     1      │ reviewer    │                     │
│  │      1      │     2      │ developer   │                     │
│  │      2      │     2      │ lead        │                     │
│  │     ...     │    ...     │ ...         │                     │
│  └─────────────┴────────────┴─────────────┘                     │
│                                                                 │
│  RELATIONSHIPS:                                                 │
│  departments ──1─────M──▶ employees    (one-to-many)           │
│  employees   ──M─────M──▶ projects     (many-to-many,          │
│                                         via assignments)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Keep this diagram open or printed. Every query in this lecture operates on this data. When you see a query, predict the result BEFORE running it."

---

# PART 3: READING DATA — SELECT

## 3.1 SELECT — Your First Query

**`SELECT` is the most important statement in SQL. You'll write it a hundred times more often than anything else.**

```sql
-- Get EVERYTHING from the employees table
SELECT * FROM employees;
```

```
 id |     name     |       email        | department_id | salary | hire_date
────┼──────────────┼────────────────────┼───────────────┼────────┼────────────
  1 │ Alice Chen   │ alice@company.com  │             1 │  95000 │ 2022-01-15
  2 │ Bob Smith    │ bob@company.com    │             1 │  85000 │ 2022-06-01
  3 │ Carol Jones  │ carol@company.com  │             2 │  75000 │ 2023-03-10
  4 │ David Lee    │ david@company.com  │             1 │ 110000 │ 2021-09-01
  5 │ Eve Wilson   │ eve@company.com    │             3 │  80000 │ 2023-01-20
  6 │ Frank Brown  │ frank@company.com  │             2 │  70000 │ 2023-07-15
  7 │ Grace Kim    │ grace@company.com  │          NULL │  90000 │ 2024-01-10
(7 rows)
```

**`SELECT *` means "give me all columns." In production, avoid it — always name the columns you need:**

```sql
-- Only the columns you care about
SELECT name, salary FROM employees;
```

```
     name     | salary
──────────────┼────────
 Alice Chen   │  95000
 Bob Smith    │  85000
 Carol Jones  │  75000
 David Lee    │ 110000
 Eve Wilson   │  80000
 Frank Brown  │  70000
 Grace Kim    │  90000
(7 rows)
```

**Column aliases with `AS` — rename columns in the output:**

```sql
SELECT
    name AS employee_name,
    salary AS annual_salary,
    salary / 12 AS monthly_salary
FROM employees;
```

```
 employee_name | annual_salary | monthly_salary
───────────────┼───────────────┼────────────────
 Alice Chen    │         95000 │           7916
 Bob Smith     │         85000 │           7083
 Carol Jones   │         75000 │           6250
 David Lee     │        110000 │           9166
 Eve Wilson    │         80000 │           6666
 Frank Brown   │         70000 │           5833
 Grace Kim     │         90000 │           7500
(7 rows)
```

> "Notice: `salary / 12` is computed per row. SQL can do arithmetic in the SELECT clause. And `AS monthly_salary` gives that computed column a name. The alias only exists in the output — it doesn't change the table."

**DISTINCT — Remove duplicate values:**

```sql
-- What departments exist in the employees table?
SELECT DISTINCT department_id FROM employees;
```

```
 department_id
───────────────
             1
             2
             3
          NULL
(4 rows)
```

---

## 3.2 WHERE — Filtering Rows

**`WHERE` is how you ask specific questions. Without it, you get everything.**

```sql
-- Engineers only (department_id = 1)
SELECT name, salary
FROM employees
WHERE department_id = 1;
```

```
    name     | salary
─────────────┼────────
 Alice Chen  │  95000
 Bob Smith   │  85000
 David Lee   │ 110000
(3 rows)
```

**Visualize what WHERE does:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     HOW WHERE WORKS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  All employees:                                                 │
│  ┌──────────────┬───────────────┐                               │
│  │     name     │ department_id │                               │
│  ├──────────────┼───────────────┤                               │
│  │ Alice Chen   │       1       │  ✅ department_id = 1         │
│  │ Bob Smith    │       1       │  ✅ department_id = 1         │
│  │ Carol Jones  │       2       │  ❌ skip                      │
│  │ David Lee    │       1       │  ✅ department_id = 1         │
│  │ Eve Wilson   │       3       │  ❌ skip                      │
│  │ Frank Brown  │       2       │  ❌ skip                      │
│  │ Grace Kim    │     NULL      │  ❌ skip                      │
│  └──────────────┴───────────────┘                               │
│                                                                 │
│  WHERE acts like a filter on every row.                         │
│  Only rows where the condition is TRUE pass through.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Operators and Conditions

**SQL gives you a rich set of operators for the WHERE clause:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   WHERE OPERATORS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  COMPARISON                                                     │
│  ├─  =          Equal to                                        │
│  ├─  != or <>   Not equal to                                    │
│  ├─  <          Less than                                       │
│  ├─  >          Greater than                                    │
│  ├─  <=         Less than or equal                              │
│  └─  >=         Greater than or equal                           │
│                                                                 │
│  RANGE                                                          │
│  └─  BETWEEN x AND y    Inclusive range                         │
│                                                                 │
│  SET MEMBERSHIP                                                 │
│  └─  IN (a, b, c)       Matches any value in the list          │
│                                                                 │
│  PATTERN MATCHING                                               │
│  ├─  LIKE 'pattern'     Case-sensitive pattern match            │
│  └─  ILIKE 'pattern'    Case-insensitive (PostgreSQL only)      │
│      Wildcards:  % = any characters,  _ = one character         │
│                                                                 │
│  NULL CHECKING                                                  │
│  ├─  IS NULL             Value is unknown/missing               │
│  └─  IS NOT NULL         Value exists                           │
│                                                                 │
│  LOGIC                                                          │
│  ├─  AND    Both must be true                                   │
│  ├─  OR     Either can be true                                  │
│  └─  NOT    Inverts the condition                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Comparison:**

```sql
-- Employees earning more than 85,000
SELECT name, salary FROM employees
WHERE salary > 85000;
```

```
    name      | salary
──────────────┼────────
 Alice Chen   │  95000
 David Lee    │ 110000
 Grace Kim    │  90000
(3 rows)
```

**BETWEEN — Inclusive range:**

```sql
-- Employees earning between 75,000 and 90,000 (inclusive)
SELECT name, salary FROM employees
WHERE salary BETWEEN 75000 AND 90000;
```

```
     name     | salary
──────────────┼────────
 Bob Smith    │  85000
 Carol Jones  │  75000
 Eve Wilson   │  80000
 Grace Kim    │  90000
(4 rows)
```

**IN — Match against a set of values:**

```sql
-- Employees in Engineering (1) or Design (3)
SELECT name, department_id FROM employees
WHERE department_id IN (1, 3);
```

```
    name      | department_id
──────────────┼───────────────
 Alice Chen   │             1
 Bob Smith    │             1
 David Lee    │             1
 Eve Wilson   │             3
(4 rows)
```

**LIKE / ILIKE — Pattern matching:**

```sql
-- Names that start with a vowel (case-insensitive)
SELECT name FROM employees
WHERE name ILIKE 'a%' OR name ILIKE 'e%';
```

```
    name
─────────────
 Alice Chen
 Eve Wilson
(2 rows)
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  LIKE WILDCARDS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  %     Matches ANY number of characters (including zero)        │
│  _     Matches exactly ONE character                            │
│                                                                 │
│  'A%'       → starts with A        (Alice, Aaron, A)           │
│  '%son'     → ends with son        (Wilson, Johnson)           │
│  '%an%'     → contains an          (Frank, Branco)             │
│  'B_b'      → B + one char + b     (Bob, Bub, Bib)            │
│  '____'     → exactly 4 characters (John, Sara, Mike)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**AND / OR / NOT — Combining conditions:**

```sql
-- Engineers earning above 90,000
SELECT name, department_id, salary FROM employees
WHERE department_id = 1 AND salary > 90000;
```

```
    name      | department_id | salary
──────────────┼───────────────┼────────
 Alice Chen   │             1 │  95000
 David Lee    │             1 │ 110000
(2 rows)
```

```sql
-- Engineers OR anyone earning above 100,000
SELECT name, department_id, salary FROM employees
WHERE department_id = 1 OR salary > 100000;
```

```
    name      | department_id | salary
──────────────┼───────────────┼────────
 Alice Chen   │             1 │  95000
 Bob Smith    │             1 │  85000
 David Lee    │             1 │ 110000
(3 rows)
```

**CRITICAL: AND has higher precedence than OR. Use parentheses!**

```sql
-- ❌ CONFUSING: What does this return?
SELECT name, department_id, salary FROM employees
WHERE department_id = 1 OR department_id = 2 AND salary > 80000;

-- This is actually parsed as:
-- WHERE department_id = 1 OR (department_id = 2 AND salary > 80000)
-- ALL of Engineering + only Marketing people above 80k
```

```sql
-- ✅ CLEAR: Use parentheses to make your intent obvious
SELECT name, department_id, salary FROM employees
WHERE (department_id = 1 OR department_id = 2) AND salary > 80000;

-- Now it's: people in Engineering OR Marketing, who also earn above 80k
```

> "Always use parentheses when mixing AND and OR. Even if you remember the precedence, the next person reading your query might not."

---

## 3.4 The NULL Trap

**This catches EVERYONE. NULL is not a value — it means "unknown."**

```sql
-- Who has no department assigned?
-- ❌ WRONG: This returns ZERO rows!
SELECT name FROM employees
WHERE department_id = NULL;
```

```
 name
──────
(0 rows)
```

> "Wait — we KNOW Grace Kim has NULL department_id. Why didn't she show up?"

**Because NULL is not equal to anything — not even itself.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE NULL TRAP                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  In Python:    None == None    → True                           │
│  In SQL:       NULL = NULL     → NULL  (not TRUE!)              │
│                                                                 │
│  NULL means "unknown." Is an unknown value equal to another     │
│  unknown value? We don't know. So the answer is... unknown.     │
│                                                                 │
│  ANY comparison with NULL produces NULL:                        │
│  ├─ NULL = NULL      → NULL                                     │
│  ├─ NULL != NULL     → NULL                                     │
│  ├─ NULL > 5         → NULL                                     │
│  ├─ NULL = 0         → NULL                                     │
│  └─ NULL != 'hello'  → NULL                                     │
│                                                                 │
│  WHERE only keeps rows where the condition is TRUE.             │
│  NULL is not TRUE. So NULL rows are filtered out.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```sql
-- ✅ CORRECT: Use IS NULL / IS NOT NULL
SELECT name, department_id FROM employees
WHERE department_id IS NULL;
```

```
   name    | department_id
───────────┼───────────────
 Grace Kim │          NULL
(1 row)
```

```sql
-- Employees who ARE assigned to a department
SELECT name, department_id FROM employees
WHERE department_id IS NOT NULL;
```

```
     name     | department_id
──────────────┼───────────────
 Alice Chen   │             1
 Bob Smith    │             1
 Carol Jones  │             2
 David Lee    │             1
 Eve Wilson   │             3
 Frank Brown  │             2
(6 rows)
```

> "Burn this into your brain: you NEVER check for NULL with `=`. Always use `IS NULL` or `IS NOT NULL`. This is not optional — `= NULL` silently returns nothing. No error, no warning. Just wrong results."

---

## 3.5 ORDER BY — Sorting Results

**Without ORDER BY, SQL returns rows in no guaranteed order. If you care about the order, you MUST specify it.**

```sql
-- Sort by salary, lowest first (ASC is the default)
SELECT name, salary FROM employees
ORDER BY salary;
```

```
     name     | salary
──────────────┼────────
 Frank Brown  │  70000
 Carol Jones  │  75000
 Eve Wilson   │  80000
 Bob Smith    │  85000
 Grace Kim    │  90000
 Alice Chen   │  95000
 David Lee    │ 110000
(7 rows)
```

```sql
-- Sort by salary, highest first
SELECT name, salary FROM employees
ORDER BY salary DESC;
```

```
     name     | salary
──────────────┼────────
 David Lee    │ 110000
 Alice Chen   │  95000
 Grace Kim    │  90000
 Bob Smith    │  85000
 Eve Wilson   │  80000
 Carol Jones  │  75000
 Frank Brown  │  70000
(7 rows)
```

**Multiple sort columns — sort by department, then by salary within each department:**

```sql
SELECT name, department_id, salary FROM employees
WHERE department_id IS NOT NULL
ORDER BY department_id ASC, salary DESC;
```

```
     name     | department_id | salary
──────────────┼───────────────┼────────
 David Lee    │             1 │ 110000
 Alice Chen   │             1 │  95000
 Bob Smith    │             1 │  85000
 Carol Jones  │             2 │  75000
 Frank Brown  │             2 │  70000
 Eve Wilson   │             3 │  80000
(6 rows)
```

> "Read it like: 'Sort by department first. Within the same department, sort by salary highest-to-lowest.' The order of columns in ORDER BY is the order of sorting priority."

**NULLS FIRST / NULLS LAST — controlling where NULLs appear:**

```sql
-- By default, NULLs sort last in ASC, first in DESC
-- Override this explicitly:
SELECT name, department_id FROM employees
ORDER BY department_id ASC NULLS FIRST;
```

```
     name     | department_id
──────────────┼───────────────
 Grace Kim    │          NULL
 Alice Chen   │             1
 Bob Smith    │             1
 David Lee    │             1
 Carol Jones  │             2
 Frank Brown  │             2
 Eve Wilson   │             3
(7 rows)
```

---

## 3.6 LIMIT and OFFSET — Pagination

**Connection to Week 4:**

> "Remember API pagination design from Week 4? You discussed offset-based and cursor-based pagination as API patterns. LIMIT and OFFSET are the SQL implementation behind offset-based pagination."

```sql
-- Give me only the first 3 employees (sorted by name)
SELECT name, salary FROM employees
ORDER BY name
LIMIT 3;
```

```
     name     | salary
──────────────┼────────
 Alice Chen   │  95000
 Bob Smith    │  85000
 Carol Jones  │  75000
(3 rows)
```

```sql
-- Page 2: Skip the first 3, give me the next 3
SELECT name, salary FROM employees
ORDER BY name
LIMIT 3 OFFSET 3;
```

```
     name     | salary
──────────────┼────────
 David Lee    │ 110000
 Eve Wilson   │  80000
 Frank Brown  │  70000
(3 rows)
```

**Mapping to API pagination:**

```
┌─────────────────────────────────────────────────────────────────┐
│               OFFSET PAGINATION (API → SQL)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  API Request:  GET /employees?page=2&page_size=3                │
│                                                                 │
│  SQL:          SELECT name, salary FROM employees               │
│                ORDER BY name                                    │
│                LIMIT 3               -- page_size               │
│                OFFSET 3;             -- (page - 1) * page_size  │
│                                                                 │
│  page=1 → OFFSET 0   (skip nothing)                            │
│  page=2 → OFFSET 3   (skip 3)                                  │
│  page=3 → OFFSET 6   (skip 6)                                  │
│                                                                 │
│  ⚠️  WARNING: OFFSET gets slower as the number grows.           │
│  Page 1000 means the database scans and throws away 3000 rows. │
│  This is why cursor-based pagination exists (Week 4, Week 6).  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Always use ORDER BY with LIMIT. Without ORDER BY, the database returns rows in arbitrary order. Two identical LIMIT queries could return different rows. ORDER BY makes pagination deterministic."

---

## 3.7 SQL Execution Order (The Hidden Truth)

**This is the most important mental model in this entire lecture.**

You WRITE SQL in this order:

```sql
SELECT   ...
FROM     ...
WHERE    ...
GROUP BY ...
HAVING   ...
ORDER BY ...
LIMIT    ...
```

But the database EXECUTES it in a DIFFERENT order:

```
┌─────────────────────────────────────────────────────────────────┐
│             SQL EXECUTION ORDER                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOU WRITE:                   DATABASE EXECUTES:                │
│  ──────────                   ──────────────────                │
│                                                                 │
│  1. SELECT columns            1. FROM    ← Which table?        │
│  2. FROM table                2. WHERE   ← Filter rows         │
│  3. WHERE condition           3. GROUP BY← Group them          │
│  4. GROUP BY column           4. HAVING  ← Filter groups       │
│  5. HAVING condition          5. SELECT  ← Pick columns        │
│  6. ORDER BY column           6. ORDER BY← Sort results        │
│  7. LIMIT n                   7. LIMIT   ← Cut to size         │
│                                                                 │
│  SELECT is near the END of execution, not the beginning!        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visualize it as a pipeline:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE SQL PIPELINE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────┐   ┌───────┐   ┌──────────┐   ┌────────┐              │
│  │ FROM │──▶│ WHERE │──▶│ GROUP BY │──▶│ HAVING │              │
│  │      │   │       │   │          │   │        │              │
│  │ All  │   │ Filter│   │ Collapse │   │ Filter │              │
│  │ rows │   │ rows  │   │ to       │   │ groups │              │
│  │      │   │       │   │ groups   │   │        │              │
│  └──────┘   └───────┘   └──────────┘   └────────┘              │
│                                                                 │
│       ┌────────┐   ┌──────────┐   ┌───────┐                    │
│  ──▶  │ SELECT │──▶│ ORDER BY │──▶│ LIMIT │  ──▶  Result       │
│       │        │   │          │   │       │                     │
│       │ Pick   │   │ Sort     │   │ Cut   │                     │
│       │ columns│   │ results  │   │ to N  │                     │
│       └────────┘   └──────────┘   └───────┘                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why does this matter? Because it explains surprising errors:**

```sql
-- ❌ ERROR: Can't use alias in WHERE (SELECT hasn't run yet!)
SELECT name, salary / 12 AS monthly
FROM employees
WHERE monthly > 7000;
-- ERROR: column "monthly" does not exist
```

```sql
-- ✅ CORRECT: Use the original expression in WHERE
SELECT name, salary / 12 AS monthly
FROM employees
WHERE salary / 12 > 7000;
```

```sql
-- ✅ ALSO WORKS: Aliases work in ORDER BY (SELECT has already run)
SELECT name, salary / 12 AS monthly
FROM employees
ORDER BY monthly DESC;
```

> "Whenever you're confused about what's 'visible' at each point, come back to this execution order. WHERE can't see aliases because SELECT hasn't happened yet. ORDER BY CAN see aliases because SELECT already ran. This is not a quirk — it's the execution pipeline."

---

# PART 4: MODIFYING DATA — INSERT, UPDATE, DELETE

## 4.1 INSERT — Adding New Rows

**You already saw INSERT during setup. Let's formalize it.**

```sql
-- Insert a single row
INSERT INTO employees (name, email, department_id, salary, hire_date)
VALUES ('Hank Porter', 'hank@company.com', 4, 78000, '2024-06-01');
```

> "Column names in parentheses tell PostgreSQL what order the values are in. You don't have to list `id` — it's SERIAL, so it auto-increments."

```sql
-- Insert multiple rows at once
INSERT INTO employees (name, email, department_id, salary, hire_date)
VALUES
    ('Iris Zhang',  'iris@company.com',  1, 92000, '2024-03-15'),
    ('Jack Davis',  'jack@company.com',  3, 73000, '2024-04-01');
```

**INSERT with DEFAULT values:**

```sql
-- Projects have DEFAULT 'planning' for status
INSERT INTO projects (name, deadline, budget)
VALUES ('Security Audit', '2024-09-01', 40000);
-- status will be 'planning' automatically
```

---

## 4.2 UPDATE — Changing Existing Rows

**UPDATE modifies existing rows.**

```sql
-- Give Alice a raise
UPDATE employees
SET salary = 100000
WHERE name = 'Alice Chen';
```

**Update multiple columns at once:**

```sql
-- Move Bob to Marketing with a salary adjustment
UPDATE employees
SET department_id = 2, salary = 88000
WHERE name = 'Bob Smith';
```

**Update based on computed values:**

```sql
-- Give everyone in Engineering a 10% raise
UPDATE employees
SET salary = salary * 1.10
WHERE department_id = 1;
```

**⚠️ THE MOST DANGEROUS MISTAKE IN SQL:**

```sql
-- ❌ CATASTROPHE: Forgot the WHERE clause!
UPDATE employees
SET salary = 0;
-- This sets EVERY employee's salary to 0. All of them. Done.
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  ⚠️  UPDATE/DELETE SAFETY RULE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ALWAYS write WHERE first.                                      │
│                                                                 │
│  Step 1: Write the WHERE condition                              │
│  Step 2: Run a SELECT with that WHERE to verify the rows        │
│  Step 3: THEN write the UPDATE/DELETE                           │
│                                                                 │
│  -- Step 1-2: Check first                                       │
│  SELECT * FROM employees WHERE name = 'Alice Chen';             │
│  -- Is this the right row? Good.                                │
│                                                                 │
│  -- Step 3: Now update                                          │
│  UPDATE employees SET salary = 100000 WHERE name = 'Alice Chen';│
│                                                                 │
│  NEVER type UPDATE or DELETE without WHERE until you've         │
│  verified the WHERE clause returns the right rows.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 DELETE — Removing Rows

**DELETE removes rows from a table.**

```sql
-- Remove a specific employee
DELETE FROM employees
WHERE name = 'Hank Porter';
```

```sql
-- Remove all completed projects
DELETE FROM projects
WHERE status = 'completed';
```

**Same danger applies:**

```sql
-- ❌ CATASTROPHE: Forgot WHERE — deletes ALL rows!
DELETE FROM employees;
-- Table is now empty. Every employee gone.
```

> "In production, you'll rarely use DELETE directly. You'll learn about 'soft deletes' in Lecture 4 this week — marking rows as deleted instead of actually removing them. But understanding hard DELETE is foundational."

---

## 4.4 The RETURNING Clause (PostgreSQL Bonus)

**PostgreSQL-specific feature: get back the data you just changed.**

```sql
-- Insert and get back the generated ID
INSERT INTO employees (name, email, department_id, salary, hire_date)
VALUES ('Kara Lin', 'kara@company.com', 3, 82000, '2024-07-01')
RETURNING id, name;
```

```
 id |   name
────┼──────────
 11 │ Kara Lin
(1 row)
```

```sql
-- Update and see the new values
UPDATE employees
SET salary = salary * 1.05
WHERE department_id = 1
RETURNING name, salary;
```

```
     name     | salary
──────────────┼────────
 Alice Chen   │ 110250
 David Lee    │ 127050
(2 rows)
```

**Connection to FastAPI:**

> "When you build an API endpoint like `POST /employees`, you create a resource and return it to the client with its generated ID. `INSERT ... RETURNING` is how you get that ID from the database in one query instead of two. You'll use this heavily when we integrate SQLAlchemy in Week 6."

Let's undo the changes we made in this section so the rest of the lecture uses consistent data:

```sql
-- Reset our data to the original state for remaining examples
-- (In practice you'd use transactions — covered in Lecture 3)
DELETE FROM employees WHERE name IN ('Hank Porter', 'Iris Zhang', 'Jack Davis', 'Kara Lin');
UPDATE employees SET salary = 95000 WHERE name = 'Alice Chen';
UPDATE employees SET department_id = 1, salary = 85000 WHERE name = 'Bob Smith';
UPDATE employees SET salary = 110000 WHERE name = 'David Lee';
DELETE FROM projects WHERE name = 'Security Audit';
INSERT INTO projects (name, status, deadline, budget) VALUES ('Marketing Site', 'completed', '2024-03-01', 30000) ON CONFLICT DO NOTHING;
```

---

# PART 5: JOINS — CONNECTING TABLES

## 5.1 The Problem — Isolated Tables

**Look at the employees table. It has `department_id = 1` for Alice. But what IS department 1?**

```sql
SELECT name, department_id FROM employees WHERE name = 'Alice Chen';
```

```
    name     | department_id
─────────────┼───────────────
 Alice Chen  │             1
```

> "Department 1. Great. What's department 1? To answer that, you need to look at the `departments` table. But SQL returns data from ONE table at a time. How do you combine them?"

**The Python way (painful):**

```python
# You'd do this in Python — two separate queries, then stitch:
employee = query("SELECT * FROM employees WHERE name = 'Alice Chen'")
department = query(f"SELECT * FROM departments WHERE id = {employee['department_id']}")
# Now combine manually...
```

**The SQL way: JOINs combine tables in a single query.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     WHY JOINS EXIST                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  In Lecture 1, you learned that normalization splits data into  │
│  separate tables to avoid redundancy.                           │
│                                                                 │
│  The tradeoff: data is spread across tables.                    │
│  JOINs are how you bring it back together for queries.          │
│                                                                 │
│  Normalization SEPARATES for storage.                           │
│  JOINs COMBINE for retrieval.                                   │
│  Two sides of the same coin.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 INNER JOIN — Only Matching Rows

**INNER JOIN returns only rows that have a match in BOTH tables.**

```sql
SELECT
    employees.name,
    employees.salary,
    departments.name AS department
FROM employees
INNER JOIN departments ON employees.department_id = departments.id;
```

```
     name     | salary | department
──────────────┼────────┼─────────────
 Alice Chen   │  95000 │ Engineering
 Bob Smith    │  85000 │ Engineering
 Carol Jones  │  75000 │ Marketing
 David Lee    │ 110000 │ Engineering
 Eve Wilson   │  80000 │ Design
 Frank Brown  │  70000 │ Marketing
(6 rows)
```

> "Six rows. But we have seven employees. Who's missing?"

Answer: **Grace Kim.** Her `department_id` is NULL. There's no department with id NULL. So she has no match, and INNER JOIN excludes her.

**Visualize INNER JOIN:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     INNER JOIN                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  employees                    departments                       │
│  ┌──────────────┬────────┐    ┌────┬─────────────┐              │
│  │     name     │ dep_id │    │ id │    name     │              │
│  ├──────────────┼────────┤    ├────┼─────────────┤              │
│  │ Alice Chen   │   1  ──┼────┼─ 1 │ Engineering │  ✅ Match    │
│  │ Bob Smith    │   1  ──┼────┼─ 1 │ Engineering │  ✅ Match    │
│  │ Carol Jones  │   2  ──┼────┼─ 2 │ Marketing   │  ✅ Match    │
│  │ David Lee    │   1  ──┼────┼─ 1 │ Engineering │  ✅ Match    │
│  │ Eve Wilson   │   3  ──┼────┼─ 3 │ Design      │  ✅ Match    │
│  │ Frank Brown  │   2  ──┼────┼─ 2 │ Marketing   │  ✅ Match    │
│  │ Grace Kim    │  NULL   │    │  4 │ Sales       │              │
│  └──────────────┴────────┘    └────┴─────────────┘              │
│       ❌ No match                  ❌ No match                   │
│       (NULL dep_id)                (No employee in Sales)       │
│                                                                 │
│  INNER JOIN = Only rows with matches on BOTH sides              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Table aliases — shorter names for readability:**

```sql
-- Instead of typing "employees" and "departments" repeatedly:
SELECT e.name, e.salary, d.name AS department
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;
-- Same result. "e" is short for "employees", "d" for "departments".
```

> "Use short, meaningful aliases. `e` for employees, `d` for departments, `p` for projects, `a` for assignments. You'll see this in every real-world SQL query."

---

## 5.3 LEFT JOIN — Keep Everything from the Left

**LEFT JOIN returns ALL rows from the left table, even if there's no match on the right. Missing right-side data becomes NULL.**

```sql
SELECT e.name, e.salary, d.name AS department
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;
```

```
     name     | salary | department
──────────────┼────────┼─────────────
 Alice Chen   │  95000 │ Engineering
 Bob Smith    │  85000 │ Engineering
 Carol Jones  │  75000 │ Marketing
 David Lee    │ 110000 │ Engineering
 Eve Wilson   │  80000 │ Design
 Frank Brown  │  70000 │ Marketing
 Grace Kim    │  90000 │ NULL
(7 rows)
```

> "All SEVEN employees now. Grace appears with NULL for department. LEFT JOIN says: 'Keep everyone from the left table (`employees`). If they have a matching department, show it. If not, fill with NULL.'"

**Visualize LEFT JOIN:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      LEFT JOIN                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  employees (LEFT)             departments (RIGHT)               │
│  ┌──────────────┬────────┐    ┌────┬─────────────┐              │
│  │     name     │ dep_id │    │ id │    name     │              │
│  ├──────────────┼────────┤    ├────┼─────────────┤              │
│  │ Alice Chen   │   1  ──┼────┼─ 1 │ Engineering │  ✅ Match    │
│  │ Bob Smith    │   1  ──┼────┼─ 1 │ Engineering │  ✅ Match    │
│  │ Carol Jones  │   2  ──┼────┼─ 2 │ Marketing   │  ✅ Match    │
│  │ David Lee    │   1  ──┼────┼─ 1 │ Engineering │  ✅ Match    │
│  │ Eve Wilson   │   3  ──┼────┼─ 3 │ Design      │  ✅ Match    │
│  │ Frank Brown  │   2  ──┼────┼─ 2 │ Marketing   │  ✅ Match    │
│  │ Grace Kim    │  NULL   │    │  4 │ Sales       │              │
│  └──────────────┴────────┘    └────┴─────────────┘              │
│       ✅ Kept! (NULL dept)         ❌ Still excluded             │
│                                    (No employee in Sales)       │
│                                                                 │
│  LEFT JOIN = ALL left rows + matching right rows                │
│  Unmatched left rows → right columns become NULL                │
│  Unmatched right rows → excluded                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Useful pattern — finding rows WITHOUT a match:**

```sql
-- Find employees NOT assigned to any department
SELECT e.name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
WHERE d.id IS NULL;
```

```
   name
───────────
 Grace Kim
(1 row)
```

> "LEFT JOIN + WHERE right.column IS NULL is a pattern for finding 'orphans' — rows in one table with no corresponding row in another. You'll use this constantly."

---

## 5.4 RIGHT JOIN and FULL JOIN

**RIGHT JOIN is the mirror of LEFT JOIN. It keeps all rows from the RIGHT table.**

```sql
SELECT e.name, d.name AS department
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id;
```

```
     name     | department
──────────────┼─────────────
 Alice Chen   │ Engineering
 Bob Smith    │ Engineering
 David Lee    │ Engineering
 Carol Jones  │ Marketing
 Frank Brown  │ Marketing
 Eve Wilson   │ Design
 NULL         │ Sales
(7 rows)
```

> "Now Sales appears (with NULL for employee name), but Grace is gone. RIGHT JOIN keeps all departments, including ones with no employees."

**In practice, RIGHT JOIN is rarely used. You can always rewrite it as LEFT JOIN by swapping the table order:**

```sql
-- These are equivalent:
-- employees RIGHT JOIN departments  =  departments LEFT JOIN employees
SELECT e.name, d.name AS department
FROM departments d
LEFT JOIN employees e ON e.department_id = d.id;
-- Same result as the RIGHT JOIN above.
```

> "Most SQL developers stick with LEFT JOIN and just change which table comes first. It's easier to read consistently."

**FULL JOIN keeps ALL rows from BOTH tables:**

```sql
SELECT e.name, d.name AS department
FROM employees e
FULL JOIN departments d ON e.department_id = d.id;
```

```
     name     | department
──────────────┼─────────────
 Alice Chen   │ Engineering
 Bob Smith    │ Engineering
 David Lee    │ Engineering
 Carol Jones  │ Marketing
 Frank Brown  │ Marketing
 Eve Wilson   │ Design
 Grace Kim    │ NULL
 NULL         │ Sales
(8 rows)
```

> "Grace (no department) AND Sales (no employees) — both appear. FULL JOIN excludes nothing."

**Summary of all four joins:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  JOIN TYPE SUMMARY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                  LEFT TABLE    RIGHT TABLE                       │
│                  (employees)   (departments)                     │
│                                                                 │
│  INNER JOIN      Only matched  Only matched    (6 rows)         │
│  LEFT JOIN       ALL           Only matched    (7 rows)         │
│  RIGHT JOIN      Only matched  ALL             (7 rows)         │
│  FULL JOIN       ALL           ALL             (8 rows)         │
│                                                                 │
│                                                                 │
│  Visual (Venn Diagram Mental Model):                            │
│                                                                 │
│  INNER:    (  [████]  )     Only the overlap                    │
│  LEFT:     ([██████]  )     All left + overlap                  │
│  RIGHT:    (  [██████])     Overlap + all right                 │
│  FULL:     ([████████])     Everything                          │
│                                                                 │
│  ⚠️  Venn diagrams are a simplification — JOINs operate on      │
│  rows, not sets of unique entities. A left table row can match  │
│  MULTIPLE right rows, producing MORE rows than either table.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 Joining Three or More Tables

**Real queries often join many tables. The key: chain JOINs one at a time.**

> "Let's answer a real question: 'Show me every employee, their department name, and the names of the projects they're working on.' This requires data from THREE tables: `employees`, `departments`, and `projects` — connected through `assignments`."

```sql
SELECT
    e.name AS employee,
    d.name AS department,
    p.name AS project,
    a.role
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
INNER JOIN assignments a ON e.id = a.employee_id
INNER JOIN projects p ON a.project_id = p.id
ORDER BY e.name, p.name;
```

```
   employee   | department  |    project     |   role
──────────────┼─────────────┼────────────────┼───────────
 Alice Chen   │ Engineering │ API Redesign   │ lead
 Alice Chen   │ Engineering │ Data Pipeline  │ developer
 Alice Chen   │ Engineering │ Mobile App     │ developer
 Bob Smith    │ Engineering │ API Redesign   │ developer
 Bob Smith    │ Engineering │ Mobile App     │ lead
 Carol Jones  │ Marketing   │ Marketing Site │ lead
 David Lee    │ Engineering │ API Redesign   │ reviewer
 David Lee    │ Engineering │ Data Pipeline  │ lead
 Eve Wilson   │ Design      │ Marketing Site │ designer
 Eve Wilson   │ Design      │ Mobile App     │ designer
 Frank Brown  │ Marketing   │ Marketing Site │ contributor
(11 rows)
```

**Walk through it step by step:**

```
┌─────────────────────────────────────────────────────────────────┐
│                MULTI-TABLE JOIN STRATEGY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Build your query ONE JOIN at a time:                           │
│                                                                 │
│  Step 1: Start with your main table                             │
│    FROM employees e                                             │
│                                                                 │
│  Step 2: Add department names                                   │
│    LEFT JOIN departments d ON e.department_id = d.id            │
│                                                                 │
│  Step 3: Connect to assignments (the junction table)            │
│    INNER JOIN assignments a ON e.id = a.employee_id             │
│                                                                 │
│  Step 4: Get project names through assignments                  │
│    INNER JOIN projects p ON a.project_id = p.id                 │
│                                                                 │
│  Each JOIN adds columns from one more table.                    │
│  The ON clause specifies HOW to match rows.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice Alice appears THREE times — once for each project she's on. JOINs can produce MORE rows than the original table when there are one-to-many or many-to-many relationships. This is expected, not a bug."

---

## 5.6 Common Join Mistakes

### Mistake 1: Forgetting the ON clause

```sql
-- ❌ WRONG: No ON clause — this creates a CROSS JOIN!
SELECT e.name, d.name
FROM employees e, departments d;
-- Returns 7 × 4 = 28 rows! Every employee paired with every department.
```

> "The old-style comma syntax (`FROM a, b`) without a WHERE condition creates a cross join — every row from table A combined with every row from table B. This is almost never what you want."

### Mistake 2: Ambiguous column names

```sql
-- ❌ ERROR: Both tables have a column called "name"
SELECT name FROM employees e
INNER JOIN departments d ON e.department_id = d.id;
-- ERROR: column reference "name" is ambiguous
```

```sql
-- ✅ CORRECT: Qualify with table alias
SELECT e.name AS employee_name, d.name AS department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;
```

### Mistake 3: Joining on the wrong column

```sql
-- ❌ WRONG: Joining on id = id (not the foreign key relationship!)
SELECT e.name, d.name
FROM employees e
INNER JOIN departments d ON e.id = d.id;
-- This matches employee.id=1 with department.id=1,
-- employee.id=2 with department.id=2... completely wrong!
```

```sql
-- ✅ CORRECT: Join on the actual foreign key
SELECT e.name, d.name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;
```

> "Always think about WHAT column connects the two tables. The foreign key in one table points to the primary key in the other. That's your ON condition."

---

# PART 6: AGGREGATION — SUMMARIZING DATA

## 6.1 Aggregate Functions

**Aggregate functions collapse many rows into one summary value.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   AGGREGATE FUNCTIONS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  COUNT(*)        Count all rows                                 │
│  COUNT(column)   Count non-NULL values in a column              │
│  SUM(column)     Add up all values                              │
│  AVG(column)     Calculate the average                          │
│  MIN(column)     Find the smallest value                        │
│  MAX(column)     Find the largest value                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```sql
-- How many employees do we have?
SELECT COUNT(*) AS total_employees FROM employees;
```

```
 total_employees
─────────────────
               7
```

```sql
-- Salary statistics
SELECT
    COUNT(*) AS total,
    SUM(salary) AS total_salary,
    AVG(salary)::INTEGER AS avg_salary,
    MIN(salary) AS lowest,
    MAX(salary) AS highest
FROM employees;
```

```
 total | total_salary | avg_salary | lowest | highest
───────┼──────────────┼────────────┼────────┼─────────
     7 │       605000 │      86428 │  70000 │  110000
```

> "The `::INTEGER` is a PostgreSQL type cast — it rounds the average to a whole number. Without it, AVG returns a decimal."

**COUNT(*) vs COUNT(column) — a subtle but important difference:**

```sql
SELECT
    COUNT(*) AS total_rows,
    COUNT(department_id) AS has_department
FROM employees;
```

```
 total_rows | has_department
────────────┼────────────────
          7 │              6
```

> "`COUNT(*)` counts ALL rows — 7. `COUNT(department_id)` counts only rows where `department_id` is NOT NULL — 6. Grace's NULL department_id is excluded from `COUNT(department_id)` but included in `COUNT(*)`."

---

## 6.2 GROUP BY — Aggregate per Category

**GROUP BY splits rows into groups, then applies the aggregate to each group separately.**

```sql
-- How many employees per department?
SELECT department_id, COUNT(*) AS employee_count
FROM employees
WHERE department_id IS NOT NULL
GROUP BY department_id;
```

```
 department_id | employee_count
───────────────┼────────────────
             1 │              3
             2 │              2
             3 │              1
```

**Visualize GROUP BY:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  HOW GROUP BY WORKS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE GROUP BY (filtered rows):                               │
│  ┌──────────────┬───────────────┐                               │
│  │     name     │ department_id │                               │
│  ├──────────────┼───────────────┤                               │
│  │ Alice Chen   │       1       │ ─┐                            │
│  │ Bob Smith    │       1       │  ├─ Group 1 (dept 1)          │
│  │ David Lee    │       1       │ ─┘                            │
│  │ Carol Jones  │       2       │ ─┐                            │
│  │ Frank Brown  │       2       │ ─┘ Group 2 (dept 2)          │
│  │ Eve Wilson   │       3       │ ── Group 3 (dept 3)          │
│  └──────────────┴───────────────┘                               │
│                                                                 │
│  AFTER GROUP BY department_id:                                  │
│  ┌───────────────┬─────────────────────┬───────────┐            │
│  │ department_id │ rows in group       │ COUNT(*)  │            │
│  ├───────────────┼─────────────────────┼───────────┤            │
│  │       1       │ Alice, Bob, David   │     3     │            │
│  │       2       │ Carol, Frank        │     2     │            │
│  │       3       │ Eve                 │     1     │            │
│  └───────────────┴─────────────────────┴───────────┘            │
│                                                                 │
│  Each group collapses into ONE output row.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The GROUP BY rule: every column in SELECT must be either in GROUP BY or inside an aggregate function.**

```sql
-- ❌ ERROR: "name" is not in GROUP BY and not aggregated
SELECT department_id, name, COUNT(*)
FROM employees
GROUP BY department_id;
-- ERROR: column "employees.name" must appear in the GROUP BY clause
-- or be used in an aggregate function
```

> "Think about it: if department 1 has Alice, Bob, and David, and the group collapses to ONE row — which name would PostgreSQL show? It can't pick one. So it refuses."

```sql
-- ✅ CORRECT: All non-aggregate columns in GROUP BY
SELECT department_id, COUNT(*) AS employee_count, AVG(salary)::INTEGER AS avg_salary
FROM employees
WHERE department_id IS NOT NULL
GROUP BY department_id
ORDER BY avg_salary DESC;
```

```
 department_id | employee_count | avg_salary
───────────────┼────────────────┼────────────
             1 │              3 │      96666
             3 │              1 │      80000
             2 │              2 │      72500
```

---

## 6.3 HAVING — Filtering Groups

**WHERE filters individual rows BEFORE grouping. HAVING filters groups AFTER grouping.**

```sql
-- Departments with more than 1 employee
SELECT department_id, COUNT(*) AS employee_count
FROM employees
WHERE department_id IS NOT NULL
GROUP BY department_id
HAVING COUNT(*) > 1;
```

```
 department_id | employee_count
───────────────┼────────────────
             1 │              3
             2 │              2
```

> "Design has only 1 employee (Eve), so it's excluded by HAVING."

**WHERE vs HAVING — a common source of confusion:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   WHERE vs HAVING                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHERE                            HAVING                        │
│  ─────                            ──────                        │
│  Filters ROWS                     Filters GROUPS                │
│  Runs BEFORE grouping             Runs AFTER grouping           │
│  Can't use aggregate functions    CAN use aggregate functions   │
│  "Which rows to consider?"        "Which groups to keep?"       │
│                                                                 │
│                                                                 │
│  Example: "Average salary by department, but only consider      │
│  employees earning above 70k, and only show departments         │
│  where the average exceeds 85k."                                │
│                                                                 │
│  SELECT department_id, AVG(salary)::INTEGER AS avg_sal          │
│  FROM employees                                                 │
│  WHERE salary > 70000            ← filter ROWS first            │
│  GROUP BY department_id                                         │
│  HAVING AVG(salary) > 85000;    ← then filter GROUPS           │
│                                                                 │
│  Pipeline:                                                      │
│  All rows → WHERE (remove low earners) → GROUP BY              │
│           → HAVING (remove low-avg departments) → result        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```sql
-- ❌ WRONG: Can't use aggregate in WHERE
SELECT department_id, AVG(salary)
FROM employees
WHERE AVG(salary) > 85000
GROUP BY department_id;
-- ERROR: aggregate functions are not allowed in WHERE
```

```sql
-- ✅ CORRECT: Use HAVING for aggregate conditions
SELECT department_id, AVG(salary)::INTEGER AS avg_salary
FROM employees
WHERE department_id IS NOT NULL
GROUP BY department_id
HAVING AVG(salary) > 85000;
```

```
 department_id | avg_salary
───────────────┼────────────
             1 │      96666
```

> "Recall the SQL execution order from section 3.7. WHERE runs second (before GROUP BY). HAVING runs fourth (after GROUP BY). Aggregates don't exist until the rows are grouped, so WHERE can't reference them. HAVING can."

---

## 6.4 Aggregation with Joins (Real-World Queries)

**The real power of SQL comes from combining JOINs and aggregation. This is how you build dashboard queries.**

```sql
-- Employee count and average salary PER DEPARTMENT NAME (not just ID)
SELECT
    d.name AS department,
    COUNT(e.id) AS employee_count,
    AVG(e.salary)::INTEGER AS avg_salary,
    SUM(e.salary) AS total_salary
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
GROUP BY d.id, d.name
ORDER BY employee_count DESC;
```

```
  department  | employee_count | avg_salary | total_salary
──────────────┼────────────────┼────────────┼──────────────
 Engineering  │              3 │      96666 │       290000
 Marketing    │              2 │      72500 │       145000
 Design       │              1 │      80000 │        80000
 Sales        │              0 │       NULL │         NULL
(4 rows)
```

> "Notice Sales shows 0 employees and NULL for avg/total salary. We used LEFT JOIN from departments, so Sales appears even with no employees. `COUNT(e.id)` counts non-NULL employee IDs — correctly returning 0 for Sales. If we'd used `COUNT(*)`, Sales would show 1 because the LEFT JOIN row itself gets counted."

```sql
-- How many projects is each employee working on?
SELECT
    e.name,
    COUNT(a.project_id) AS project_count
FROM employees e
LEFT JOIN assignments a ON e.id = a.employee_id
GROUP BY e.id, e.name
ORDER BY project_count DESC;
```

```
     name     | project_count
──────────────┼───────────────
 Alice Chen   │             3
 Bob Smith    │             2
 David Lee    │             2
 Eve Wilson   │             2
 Carol Jones  │             1
 Frank Brown  │             1
 Grace Kim    │             0
(7 rows)
```

> "Grace has 0 projects — the LEFT JOIN preserves her, and `COUNT(a.project_id)` correctly counts zero non-NULL values. Alice is on 3 projects, the busiest."

---

# PART 7: SUBQUERIES AND CTEs

## 7.1 Subqueries — Queries Inside Queries

**A subquery is a SELECT inside another SELECT. It's a query that feeds into a larger query.**

**Subquery in WHERE (most common):**

```sql
-- Employees earning above the company average
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

```
    name      | salary
──────────────┼────────
 Alice Chen   │  95000
 David Lee    │ 110000
 Grace Kim    │  90000
(3 rows)
```

> "The inner query `(SELECT AVG(salary) FROM employees)` returns a single value: 86428. The outer query uses that value in the WHERE clause. PostgreSQL executes the inner query first, gets the number, then plugs it into the outer query."

**Subquery with IN (returning multiple values):**

```sql
-- Employees who are project leads
SELECT name FROM employees
WHERE id IN (
    SELECT employee_id FROM assignments
    WHERE role = 'lead'
);
```

```
     name
──────────────
 Alice Chen
 Bob Smith
 Carol Jones
 David Lee
(4 rows)
```

**Subquery in FROM (derived table):**

```sql
-- Department salary stats, then filter by stats
SELECT department, avg_salary
FROM (
    SELECT
        d.name AS department,
        AVG(e.salary)::INTEGER AS avg_salary
    FROM departments d
    INNER JOIN employees e ON d.id = e.department_id
    GROUP BY d.id, d.name
) AS dept_stats
WHERE avg_salary > 75000;
```

```
  department  | avg_salary
──────────────┼────────────
 Engineering  │      96666
 Design       │      80000
(2 rows)
```

> "The inner query calculates per-department stats. The outer query filters those stats. The `AS dept_stats` gives a name to the derived table — PostgreSQL requires this."

---

## 7.2 CTEs — The WITH Clause

**CTEs (Common Table Expressions) solve the same problem as subqueries but with a dramatic improvement in readability.**

```sql
-- Same query as above, but written as a CTE
WITH dept_stats AS (
    SELECT
        d.name AS department,
        AVG(e.salary)::INTEGER AS avg_salary
    FROM departments d
    INNER JOIN employees e ON d.id = e.department_id
    GROUP BY d.id, d.name
)
SELECT department, avg_salary
FROM dept_stats
WHERE avg_salary > 75000;
```

> "Read the CTE as: 'WITH this named result set (`dept_stats`), now do this query...' It reads top-to-bottom. The subquery version forces you to read inside-out."

**Chaining multiple CTEs:**

```sql
-- Complex analysis: employees, their project count, and department average
WITH project_counts AS (
    SELECT
        employee_id,
        COUNT(*) AS num_projects
    FROM assignments
    GROUP BY employee_id
),
dept_averages AS (
    SELECT
        department_id,
        AVG(salary)::INTEGER AS dept_avg_salary
    FROM employees
    WHERE department_id IS NOT NULL
    GROUP BY department_id
)
SELECT
    e.name,
    d.name AS department,
    e.salary,
    da.dept_avg_salary,
    COALESCE(pc.num_projects, 0) AS projects
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
LEFT JOIN dept_averages da ON e.department_id = da.department_id
LEFT JOIN project_counts pc ON e.id = pc.employee_id
ORDER BY projects DESC;
```

```
     name     | department  | salary | dept_avg_salary | projects
──────────────┼─────────────┼────────┼─────────────────┼──────────
 Alice Chen   │ Engineering │  95000 │           96666 │        3
 Bob Smith    │ Engineering │  85000 │           96666 │        2
 David Lee    │ Engineering │ 110000 │           96666 │        2
 Eve Wilson   │ Design      │  80000 │           80000 │        2
 Carol Jones  │ Marketing   │  75000 │           72500 │        1
 Frank Brown  │ Marketing   │  70000 │           72500 │        1
 Grace Kim    │ NULL        │  90000 │            NULL │        0
(7 rows)
```

> "Notice `COALESCE(pc.num_projects, 0)` — this replaces NULL with 0. Grace has no assignments, so the LEFT JOIN produces NULL for `num_projects`. COALESCE says 'if it's NULL, use 0 instead.' Think of it like Python's `value if value is not None else 0`."

---

## 7.3 When to Use Each

```
┌─────────────────────────────────────────────────────────────────┐
│              SUBQUERY vs CTE: WHEN TO USE WHICH                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE A SUBQUERY WHEN:                                           │
│  ├─ It's simple and short (one line, one value)                 │
│  ├─ It's used only once                                         │
│  └─ WHERE x IN (SELECT ...) or WHERE x > (SELECT ...)          │
│                                                                 │
│  USE A CTE WHEN:                                                │
│  ├─ The logic is complex or multi-step                          │
│  ├─ You reference the result more than once                     │
│  ├─ You want readable, top-to-bottom flow                       │
│  └─ You're building a query piece by piece (debugging easier)   │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  If a subquery makes your eyes cross → rewrite as a CTE.       │
│  CTEs read top-to-bottom. Subqueries read inside-out.           │
│  Top-to-bottom wins for maintainability.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                    SQL QUICK REFERENCE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  READ DATA:                                                     │
│      SELECT columns FROM table                                  │
│      WHERE condition                                            │
│      ORDER BY column [ASC|DESC]                                 │
│      LIMIT n OFFSET m;                                          │
│                                                                 │
│  ADD DATA:                                                      │
│      INSERT INTO table (columns) VALUES (values);               │
│                                                                 │
│  CHANGE DATA:                                                   │
│      UPDATE table SET col = val WHERE condition;                │
│      ⚠️  Always include WHERE!                                   │
│                                                                 │
│  REMOVE DATA:                                                   │
│      DELETE FROM table WHERE condition;                         │
│      ⚠️  Always include WHERE!                                   │
│                                                                 │
│  COMBINE TABLES:                                                │
│      SELECT ... FROM a INNER JOIN b ON a.fk = b.pk             │
│      SELECT ... FROM a LEFT JOIN b ON a.fk = b.pk              │
│                                                                 │
│  SUMMARIZE:                                                     │
│      SELECT col, COUNT(*) FROM table                            │
│      GROUP BY col                                               │
│      HAVING COUNT(*) > n;                                       │
│                                                                 │
│  COMPOSE:                                                       │
│      WITH name AS (SELECT ...) SELECT ... FROM name;            │
│                                                                 │
│  KEY OPERATORS:                                                 │
│      =  !=  <  >  <=  >=                                        │
│      BETWEEN x AND y                                            │
│      IN (a, b, c)                                               │
│      LIKE '%pattern%'                                           │
│      IS NULL / IS NOT NULL                                      │
│      AND  OR  NOT                                               │
│                                                                 │
│  EXECUTION ORDER (memorize this):                               │
│      FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ WHERE col = NULL      → Use IS NULL                     │
│      ❌ UPDATE without WHERE  → Changes ALL rows                │
│      ❌ DELETE without WHERE  → Deletes ALL rows                │
│      ❌ Alias in WHERE        → Use original expression         │
│      ❌ COUNT(*) with LEFT JOIN → Use COUNT(column)             │
│      ❌ Aggregate in WHERE    → Use HAVING                      │
│                                                                 │
│  psql SHORTCUTS:                                                │
│      \dt  List tables    \d table  Describe table               │
│      \l   List databases \q        Quit                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  SQL = DECLARATIVE DATA QUESTIONS                               │
│                                                                 │
│  You describe the RESULT you want.                              │
│  PostgreSQL figures out HOW to get it.                           │
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐     │
│  │   FROM   │──▶│  WHERE   │──▶│ GROUP BY │──▶│  SELECT  │     │
│  │          │   │          │   │          │   │          │     │
│  │ Pick the │   │ Filter   │   │ Collapse │   │ Shape    │     │
│  │ table(s) │   │ rows     │   │ groups   │   │ output   │     │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘     │
│                                                                 │
│  THE IMPERATIVE → DECLARATIVE SHIFT:                            │
│  ├─ Python: "Loop through rows, check each, collect matches"    │
│  ├─ SQL:    "Give me rows WHERE this condition is true"         │
│  └─ Same result. Different way of thinking.                     │
│                                                                 │
│  THE JOIN MENTAL MODEL:                                         │
│  ├─ INNER JOIN  = "Only matching rows from both sides"          │
│  ├─ LEFT JOIN   = "All rows from left, matches from right"      │
│  ├─ RIGHT JOIN  = "Matches from left, all rows from right"      │
│  └─ FULL JOIN   = "Everything from both sides"                  │
│                                                                 │
│  THE AGGREGATION MODEL:                                         │
│  ├─ GROUP BY    = "Split rows into buckets"                     │
│  ├─ COUNT/SUM   = "Summarize each bucket"                       │
│  ├─ WHERE       = "Filter rows BEFORE bucketing"                │
│  └─ HAVING      = "Filter buckets AFTER summarizing"            │
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
│  WEEK 5, LECTURE 3: PostgreSQL-Specific Features                │
│  └─ JSONB columns, full-text search, EXPLAIN basics,            │
│     indexes (why some queries are fast and others slow),        │
│     and transactions (BEGIN, COMMIT, ROLLBACK).                 │
│     You'll learn why ORDER + LIMIT can be instant on            │
│     1 million rows — if there's an index.                       │
│                                                                 │
│  WEEK 5, LECTURE 4: Database Design Workshop                    │
│  └─ Apply everything: design a real schema from scratch,        │
│     write the CREATE TABLEs, populate data, query it.           │
│     Hands-on practice with the SQL you just learned.            │
│                                                                 │
│  WEEK 5, MINI-PROJECT: Design and Populate a Database           │
│  └─ You'll design a multi-entity schema, write raw SQL          │
│     queries, and document it with an ER diagram.                │
│     Everything from this lecture applied to YOUR design.        │
│                                                                 │
│  WEEK 6: SQLAlchemy & Alembic                                   │
│  └─ You'll express these same queries in Python using           │
│     an ORM. Understanding raw SQL first means you'll            │
│     KNOW what the ORM is generating under the hood.             │
│     select(Employee).where(Employee.salary > 90000)             │
│     is just SQL with a Python accent.                           │
│                                                                 │
│  WEEK 7: Query Optimization                                     │
│  └─ EXPLAIN ANALYZE on the queries you write today.             │
│     You'll see WHY a query is slow and HOW indexes fix it.      │
│     You can't optimize what you don't understand.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```