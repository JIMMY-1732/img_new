# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "Database design is about storing data so my application can INSERT and SELECT what it needs. If queries return results and nothing crashes, the schema is fine. Relationships are just matching integer IDs. Constraints are optional safety nets I'll add if I have time."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | ON DELETE CASCADE only removes the immediate children of a deleted row | CASCADE chains transitively through every FK that specifies it — deleting a grandparent can wipe four levels of descendants |
| 2 | A UNIQUE constraint prevents duplicate NULL values the same way it prevents duplicate non-NULL values | In SQL, NULL ≠ NULL. UNIQUE treats two NULLs as *distinct*, so multiple rows with the same non-NULL column and a NULL column coexist silently |
| 3 | JOINing to a related table's current value gives you the historical value at the time of the event | JOIN always reads *current state*. If a product price changes, every past order total retroactively changes when queried via JOIN |
| 4 | Soft-deleted rows are invisible to UNIQUE constraints | A standard UNIQUE index sees ALL rows. Soft-deleting a row does not free its unique slot for a new row with the same value |
| 5 | Fewer tables is simpler and therefore better | Fewer tables means more redundancy, more anomalies, and more categories of bugs that the database *cannot prevent* |
| 6 | Application-level validation is sufficient to enforce business rules | Application code has bugs, race conditions, and multiple entry points. Database constraints are absolute — they cannot be bypassed |
| 7 | RESTRICT means you can never delete the parent row | RESTRICT prevents deletion *while* children exist. Remove or reassign children first, then the parent deletes fine |

**The Prediction Gap:** When students see a schema with ON DELETE CASCADE on three levels of foreign keys and are asked "what happens when you delete the top-level row?", they predict only one level of children is removed. The reality is that CASCADE propagates through every level, potentially destroying hundreds of rows across multiple tables from a single DELETE statement.

---

## The Main Purpose (Essence)

**Core Insight:** Database schema design is the practice of encoding business invariants into structural constraints so that **invalid data cannot exist** — not so that data is stored conveniently. The database is the last line of defense; application bugs cannot violate constraints enforced at the schema level.

**The Problems Being Solved:**

| Design Decision | Without It | With It |
|----------------|-----------|---------|
| Foreign Keys with REFERENCES | Orphaned rows, dangling IDs, silent data corruption | Database rejects invalid references; relationships guaranteed |
| ON DELETE behavior (CASCADE/RESTRICT/SET NULL) | Accidental deletion of financial records or orphaned children | Deletion behavior is explicit, documented, and enforced |
| Snapshot data (unit_price on order_items) | Historical records retroactively change when source data updates | Point-in-time accuracy preserved forever |
| Soft delete (deleted_at) | Deleting referenced rows breaks FKs or loses audit history | Data hidden but preserved; references intact; recoverable |
| Partial unique indexes | Soft-deleted rows block re-insertion of the same unique value | Only active rows participate in uniqueness checks |

**The Key Distinction:**
- **Application validation** = "Please don't insert bad data" (polite request, bypassable)
- **Database constraints** = "You CANNOT insert bad data" (structural guarantee, absolute)

A well-designed schema makes entire *categories* of bugs impossible. No amount of application testing achieves the same guarantee.

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "CASCADE only removes direct children" | Deleting a department removes its projects only | Deleting a department also destroys all tasks AND assignments (3 levels deep) | CASCADE chains transitively through every FK with ON DELETE CASCADE |
| "NULL = NULL in comparisons" | UNIQUE(employee_id, project_id) prevents two (1, NULL) rows | Both rows coexist — no violation raised | SQL spec: NULL ≠ NULL for UNIQUE constraint evaluation |
| "JOIN returns the value at time of the event" | Alice's January order shows $29.99 (original price) | Alice's January order shows $39.99 (current price) | JOIN reads live table state, not historical snapshots |
| "Soft-deleted rows don't count for UNIQUE" | Re-inserting "Widget" after soft-deleting it succeeds | ERROR: unique constraint violation | Standard UNIQUE indexes include all rows regardless of deleted_at |
| "RESTRICT means permanent lock" | Parent row can never be deleted | Delete children first, then parent deletes normally | RESTRICT blocks deletion only while FK references exist |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Cascade Chain Reaction

**Context:** A company reorganizes its engineering department. A manager runs a cleanup script to remove the old department. The following schema is in production.

```sql
-- Setup: Run this in your PostgreSQL instance
CREATE TABLE departments (
    id   BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE projects (
    id      BIGSERIAL PRIMARY KEY,
    dept_id BIGINT NOT NULL REFERENCES departments(id) ON DELETE CASCADE,
    name    TEXT NOT NULL
);

CREATE TABLE tasks (
    id         BIGSERIAL PRIMARY KEY,
    project_id BIGINT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    title      TEXT NOT NULL,
    status     TEXT NOT NULL DEFAULT 'open'
);

CREATE TABLE time_entries (
    id          BIGSERIAL PRIMARY KEY,
    task_id     BIGINT NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    employee    TEXT NOT NULL,
    hours       NUMERIC(5,2) NOT NULL,
    logged_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed data: 1 department, 2 projects, 3 tasks, 5 time entries
INSERT INTO departments VALUES (1, 'Engineering');
INSERT INTO projects VALUES (1, 1, 'API Rewrite'), (2, 1, 'Mobile App');
INSERT INTO tasks VALUES (1, 1, 'Design Schema', 'done'), 
                         (2, 1, 'Write Tests', 'open'),
                         (3, 2, 'Build Login Screen', 'open');
INSERT INTO time_entries VALUES 
    (1, 1, 'Alice', 4.5, '2025-03-01'),
    (2, 1, 'Bob', 3.0, '2025-03-02'),
    (3, 2, 'Alice', 6.0, '2025-03-03'),
    (4, 3, 'Charlie', 8.0, '2025-03-04'),
    (5, 3, 'Charlie', 2.5, '2025-03-05');
```

**Now the manager runs:**

```sql
DELETE FROM departments WHERE id = 1;
```

**Then queries:**

```sql
SELECT 'departments' AS tbl, COUNT(*) FROM departments
UNION ALL
SELECT 'projects', COUNT(*) FROM projects
UNION ALL
SELECT 'tasks', COUNT(*) FROM tasks
UNION ALL
SELECT 'time_entries', COUNT(*) FROM time_entries;
```

**Questions:**

1. **Predict the output.** Write the exact counts for each table. How many rows survive in each?

2. **Explain why.** A developer expected only the department row and its two project rows to be deleted (3 rows total). They are shocked to discover that 11 rows disappeared. Explain the mechanism that caused this.

3. **Mental Trace:** Walk through each step PostgreSQL performs after `DELETE FROM departments WHERE id = 1`. For each table, state what PostgreSQL checks, what it finds, and what action it takes. How many total DELETE operations does PostgreSQL perform?

---

## Exercise 1.2: The Duplicate That Isn't

**Context:** A team is building an employee assignment system. Employees are assigned to projects, but some are "on the bench" (no project). The schema uses a UNIQUE constraint to prevent duplicate assignments.

```sql
CREATE TABLE team_assignments (
    id          BIGSERIAL PRIMARY KEY,
    employee_id BIGINT NOT NULL,
    project_id  BIGINT,          -- NULL means "on bench"
    role        TEXT NOT NULL,
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(employee_id, project_id)
);

-- Alice is assigned to Project 10
INSERT INTO team_assignments (employee_id, project_id, role) 
VALUES (1, 10, 'developer');

-- Alice is also assigned to Project 20 (different project — should work)
INSERT INTO team_assignments (employee_id, project_id, role) 
VALUES (1, 20, 'reviewer');

-- Bob goes on bench (no project)
INSERT INTO team_assignments (employee_id, project_id, role) 
VALUES (2, NULL, 'available');

-- A system bug accidentally tries to put Bob on bench AGAIN
INSERT INTO team_assignments (employee_id, project_id, role) 
VALUES (2, NULL, 'available');
```

**Questions:**

1. **Predict the outcome.** Does the fourth INSERT (duplicate bench assignment for Bob) succeed or fail? How many rows are in the table after all four statements?

2. **Explain why.** The developer who designed this constraint expected it to prevent Bob from having two "on bench" entries. Why does the constraint not catch this? What rule of SQL is at play?

3. **Mental Trace:** When PostgreSQL evaluates the UNIQUE constraint for the fourth INSERT, it compares `(2, NULL)` against the existing `(2, NULL)`. Walk through the comparison: what does `NULL = NULL` evaluate to in SQL? How does the UNIQUE constraint interpret this result?

---

## Exercise 1.3: The Time-Traveling Invoice

**Context:** A bookstore tracks purchases by JOINing to the current book price. A developer argues this is "clean" because the price only lives in one place (no duplication).

```sql
CREATE TABLE books (
    id    BIGSERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    price NUMERIC(10,2) NOT NULL
);

CREATE TABLE purchases (
    id          BIGSERIAL PRIMARY KEY,
    customer    TEXT NOT NULL,
    book_id     BIGINT NOT NULL REFERENCES books(id),
    quantity    INTEGER NOT NULL,
    purchased_at TIMESTAMPTZ NOT NULL DEFAULT now()
    -- Note: NO price column here — "we'll just JOIN to books"
);

-- January: Book costs $29.99
INSERT INTO books VALUES (1, 'Database Design in Practice', 29.99);
INSERT INTO purchases VALUES (1, 'Alice', 1, 3, '2025-01-15');
INSERT INTO purchases VALUES (2, 'Bob', 1, 1, '2025-01-20');

-- March: Publisher raises the price
UPDATE books SET price = 44.99 WHERE id = 1;

-- April: Carol buys the book at the new price
INSERT INTO purchases VALUES (3, 'Carol', 1, 2, '2025-04-10');
```

**Now the accounting department runs the invoice report:**

```sql
SELECT 
    p.customer,
    p.purchased_at::DATE AS purchase_date,
    b.title,
    b.price AS unit_price,
    p.quantity,
    (p.quantity * b.price) AS line_total
FROM purchases p
JOIN books b ON b.id = p.book_id
ORDER BY p.purchased_at;
```

**Questions:**

1. **Predict the output.** Write the unit_price and line_total for each of the three customers. Pay attention to whether Alice and Bob (who bought in January at $29.99) show the original or current price.

2. **Explain why.** The accounting department complains: "Alice's invoice from January shows $44.99 per book, but she paid $29.99. Our records are wrong." Why does this happen? What fundamental misunderstanding led to this schema design?

3. **Mental Trace:** When the SELECT...JOIN executes, trace what value `b.price` resolves to for each row. Does the JOIN have any concept of "what the price was at `purchased_at` time"? Why or why not?

---

## Exercise 1.4: The Ghost in the Constraint

**Context:** A music streaming service uses soft deletes for playlists. When a user "deletes" a playlist, it's hidden but preserved. The developer has added a UNIQUE constraint to ensure playlist names are unique per user.

```sql
CREATE TABLE playlists (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    name        TEXT NOT NULL,
    deleted_at  TIMESTAMPTZ,          -- NULL = active, non-NULL = soft-deleted
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, name)             -- Standard UNIQUE constraint
);

-- Alice creates "Workout Mix"
INSERT INTO playlists (user_id, name) VALUES (1, 'Workout Mix');

-- Alice soft-deletes it (tired of that playlist)
UPDATE playlists SET deleted_at = now() WHERE user_id = 1 AND name = 'Workout Mix';

-- A month later, Alice creates a NEW "Workout Mix" (fresh start!)
INSERT INTO playlists (user_id, name) VALUES (1, 'Workout Mix');
```

**Questions:**

1. **Predict the outcome.** Does the third statement (re-creating "Workout Mix") succeed or fail? If it fails, what error does PostgreSQL raise?

2. **Explain why.** Alice's original playlist is soft-deleted. She can't see it in the UI. From her perspective, the name "Workout Mix" is available. Why does the database disagree?

3. **Mental Trace:** When PostgreSQL evaluates the UNIQUE constraint for the new INSERT, what rows does the standard UNIQUE index see? Does it distinguish between active and soft-deleted rows? What would need to change in the schema to allow this INSERT while still preventing *active* duplicate names?

---

# Level 1 Solutions

## Solution 1.1: The Cascade Chain Reaction

**1. Predicted Output:**

```
 tbl          | count
--------------+-------
 departments  |     0
 projects     |     0
 tasks        |     0
 time_entries |     0
```

Every single row across all four tables is deleted. **11 rows destroyed** by one DELETE statement.

**2. Explain Why:**

ON DELETE CASCADE is *transitive*. It doesn't stop at the first level — it propagates through every FK relationship that specifies CASCADE:

- Deleting department 1 triggers CASCADE on the `projects.dept_id` FK → both projects are deleted.
- Deleting projects 1 and 2 triggers CASCADE on the `tasks.project_id` FK → all 3 tasks are deleted.
- Deleting tasks 1, 2, and 3 triggers CASCADE on the `time_entries.task_id` FK → all 5 time entries are deleted.

The developer's mental model was that CASCADE is "one hop." The reality is that CASCADE follows every FK chain to its terminus, regardless of depth.

**3. Mental Trace:**

```
1. DELETE FROM departments WHERE id = 1
   → PostgreSQL deletes department row (id=1)
   → Checks: "Any FK references departments(id) with ON DELETE CASCADE?"
   → Found: projects.dept_id → CASCADE

2. PostgreSQL deletes projects WHERE dept_id = 1
   → Deletes project 1 (API Rewrite) and project 2 (Mobile App)
   → Checks: "Any FK references projects(id) with ON DELETE CASCADE?"
   → Found: tasks.project_id → CASCADE

3. PostgreSQL deletes tasks WHERE project_id IN (1, 2)
   → Deletes task 1, 2 (project 1) and task 3 (project 2)
   → Checks: "Any FK references tasks(id) with ON DELETE CASCADE?"
   → Found: time_entries.task_id → CASCADE

4. PostgreSQL deletes time_entries WHERE task_id IN (1, 2, 3)
   → Deletes all 5 time entries
   → Checks: "Any FK references time_entries(id) with CASCADE?"
   → None found. Chain terminates.

Total operations: 1 + 2 + 3 + 5 = 11 rows deleted.
```

**The Key Insight:**
CASCADE chains are invisible in the schema unless you trace every FK. In a production system, this chain could destroy years of time-tracking data from a single department deletion. This is exactly why the lecture warns: "CASCADE — use when children are meaningless without parent." Time entries have independent business value (payroll, billing) — they are NOT meaningless without their department. The correct choice here is RESTRICT on most of these FKs, combined with soft deletes.

---

## Solution 1.2: The Duplicate That Isn't

**1. Predicted Outcome:**

The fourth INSERT **succeeds**. The table has **4 rows**. Bob has two "on bench" assignments.

**2. Explain Why:**

In SQL, the expression `NULL = NULL` does not evaluate to `TRUE` — it evaluates to `UNKNOWN`. PostgreSQL's UNIQUE constraint considers two values "duplicate" only if their comparison returns `TRUE`. Since `NULL = NULL` returns `UNKNOWN`, PostgreSQL treats `(2, NULL)` and `(2, NULL)` as *distinct* values. The UNIQUE constraint is not violated.

This is the SQL standard behavior (not a PostgreSQL quirk). NULL means "unknown" — and two unknown values are not proven to be the same.

**3. Mental Trace:**

```
1. Fourth INSERT attempts: (employee_id=2, project_id=NULL)
2. PostgreSQL checks UNIQUE(employee_id, project_id)
3. Existing row found: (employee_id=2, project_id=NULL)
4. Comparison: employee_id: 2 = 2 → TRUE
5. Comparison: project_id: NULL = NULL → UNKNOWN
6. Combined: TRUE AND UNKNOWN → UNKNOWN
7. UNIQUE violation requires TRUE (proven duplicate)
8. UNKNOWN ≠ TRUE → No violation detected
9. INSERT succeeds. Bob now has two bench entries.
```

**The Key Insight:**
If a column participating in a UNIQUE constraint can be NULL, the constraint cannot prevent duplicate NULLs. To enforce "one bench assignment per employee," you need a partial unique index:

```sql
-- Remove the table-level UNIQUE constraint, then:
CREATE UNIQUE INDEX idx_one_bench_per_employee
ON team_assignments (employee_id)
WHERE project_id IS NULL;

-- This index ONLY covers rows where project_id IS NULL,
-- and enforces that each employee_id appears at most once
-- among those rows.
```

This is the same technique the lecture teaches for soft-delete uniqueness — partial indexes that scope constraints to a subset of rows.

---

## Solution 1.3: The Time-Traveling Invoice

**1. Predicted Output:**

```
 customer | purchase_date | title                        | unit_price | quantity | line_total
----------|---------------|------------------------------|------------|----------|----------
 Alice    | 2025-01-15    | Database Design in Practice  |      44.99 |        3 |    134.97
 Bob      | 2025-01-20    | Database Design in Practice  |      44.99 |        1 |     44.99
 Carol    | 2025-04-10    | Database Design in Practice  |      44.99 |        2 |     89.98
```

**ALL THREE customers show $44.99**, including Alice and Bob who purchased in January at $29.99.

**2. Explain Why:**

The `JOIN books b ON b.id = p.book_id` reads the *current* row in the books table. There is only one row for book id=1, and its price is now 44.99. The JOIN has no concept of "what was the price on January 15th?" It simply looks up the book's current state.

The fundamental misunderstanding: the developer confused "avoiding data duplication" (a valid normalization goal) with "never storing the same fact twice." In this case, `unit_price` at the time of purchase is a *different fact* from the book's current price. They happen to have the same value initially, but they diverge over time. This is the Snapshot Data pattern from the lecture — some values must be captured at the time of the event because the source can change.

**3. Mental Trace:**

```
For each row in purchases:
  1. p = {customer: 'Alice', book_id: 1, quantity: 3, purchased_at: '2025-01-15'}
  2. JOIN books b ON b.id = 1
  3. PostgreSQL reads books WHERE id = 1 → {price: 44.99}  ← CURRENT value!
  4. b.price resolves to 44.99 regardless of when the purchase occurred
  5. line_total = 3 × 44.99 = 134.97

The JOIN does not accept a temporal parameter.
"books.price as of 2025-01-15" is not a concept the JOIN understands.
The price column stores exactly ONE value: the most recent one.
```

**The Fix:**

```sql
CREATE TABLE purchases (
    id           BIGSERIAL PRIMARY KEY,
    customer     TEXT NOT NULL,
    book_id      BIGINT NOT NULL REFERENCES books(id),
    quantity     INTEGER NOT NULL,
    unit_price   NUMERIC(10,2) NOT NULL,   -- SNAPSHOT: frozen at purchase time
    purchased_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Insert captures price at time of purchase:
INSERT INTO purchases (customer, book_id, quantity, unit_price, purchased_at)
VALUES ('Alice', 1, 3, 29.99, '2025-01-15');
-- Now Alice's unit_price is 29.99 FOREVER, regardless of books.price changes.
```

---

## Solution 1.4: The Ghost in the Constraint

**1. Predicted Outcome:**

The third statement **fails** with:

```
ERROR: duplicate key value violates unique constraint "playlists_user_id_name_key"
DETAIL: Key (user_id, name)=(1, Workout Mix) already exists.
```

**2. Explain Why:**

The soft-deleted row still physically exists in the table. Its `deleted_at` column is non-NULL, but the standard UNIQUE index on `(user_id, name)` does not examine `deleted_at`. It sees ALL rows — active and soft-deleted alike. The row `(user_id=1, name='Workout Mix')` occupies the unique slot regardless of its deletion status.

From Alice's perspective (through the application UI that filters `WHERE deleted_at IS NULL`), the name is available. But from PostgreSQL's perspective, the row is still there.

**3. Mental Trace:**

```
1. Third INSERT attempts: (user_id=1, name='Workout Mix')
2. PostgreSQL checks UNIQUE(user_id, name)
3. Scans the UNIQUE index for (1, 'Workout Mix')
4. Finds existing row: id=1, user_id=1, name='Workout Mix', deleted_at='2025-06-15...'
5. Standard UNIQUE index does NOT check deleted_at
6. Match found → UNIQUE violation → INSERT rejected
```

**The Fix:** Replace the standard UNIQUE constraint with a partial unique index that only considers active rows:

```sql
-- 1. Remove the table-level constraint
-- (In practice, recreate the table or ALTER TABLE DROP CONSTRAINT)

-- 2. Add a partial unique index
CREATE UNIQUE INDEX idx_playlists_unique_active_name
ON playlists (user_id, name)
WHERE deleted_at IS NULL;
```

Now only rows where `deleted_at IS NULL` participate in the uniqueness check. The soft-deleted "Workout Mix" is excluded, and Alice's new INSERT succeeds — while still preventing two *active* playlists with the same name.

This is the exact pattern the lecture teaches in Part 4 for soft deletes. The exercise demonstrates *why* it's necessary by letting you feel the failure first.

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Price Chameleon

**Context:** A colleague designed the schema for an online course platform. Students purchase course bundles. The system has been in production for 3 months. The finance team just reported that Q1 revenue numbers are wrong — refund amounts don't match what customers originally paid.

```sql
CREATE TABLE courses (
    id          BIGSERIAL PRIMARY KEY,
    title       TEXT NOT NULL,
    price       NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE students (
    id         BIGSERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    email      TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE enrollments (
    id           BIGSERIAL PRIMARY KEY,
    student_id   BIGINT NOT NULL REFERENCES students(id),
    course_id    BIGINT NOT NULL REFERENCES courses(id),
    enrolled_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Developer's note: "price lives in courses table, no need to duplicate"
    UNIQUE(student_id, course_id)
);

-- Seed data
INSERT INTO courses VALUES (1, 'Python Basics', 49.99, now(), now());
INSERT INTO courses VALUES (2, 'SQL Mastery', 79.99, now(), now());

INSERT INTO students VALUES (1, 'Alice', 'alice@mail.com', now());
INSERT INTO students VALUES (2, 'Bob', 'bob@mail.com', now());

-- January enrollments (at original prices)
INSERT INTO enrollments (student_id, course_id, enrolled_at) 
VALUES (1, 1, '2025-01-10');  -- Alice buys Python Basics at $49.99
INSERT INTO enrollments (student_id, course_id, enrolled_at)
VALUES (1, 2, '2025-01-15');  -- Alice buys SQL Mastery at $79.99
INSERT INTO enrollments (student_id, course_id, enrolled_at)
VALUES (2, 1, '2025-01-20');  -- Bob buys Python Basics at $49.99

-- February: Prices change for spring sale
UPDATE courses SET price = 29.99, updated_at = now() WHERE id = 1;  -- Python Basics: $49.99 → $29.99
UPDATE courses SET price = 59.99, updated_at = now() WHERE id = 2;  -- SQL Mastery: $79.99 → $59.99
```

**The revenue report query:**

```sql
SELECT 
    s.name AS student,
    c.title AS course,
    e.enrolled_at::DATE AS enrollment_date,
    c.price AS amount_paid
FROM enrollments e
JOIN students s ON s.id = e.student_id
JOIN courses c ON c.id = e.course_id
ORDER BY e.enrolled_at;
```

---

**Task 2a: Identify the Flaw**

Run the revenue report query AFTER the price updates.

1. What amount does it show for Alice's January enrollment in Python Basics?
2. What is the actual amount Alice paid?
3. What is the total revenue reported vs. the actual total revenue?

Do NOT fix the code yet. Identify and articulate the precise flaw.

---

**Task 2b: Explain the Mechanism**

1. Why does the `enrollments` table fail to preserve historical pricing?
2. The developer argued "normalization says don't duplicate data." Why is this argument wrong in this specific case?
3. What category of data from the lecture does this fall into?

---

**Task 2c: Instrument to Prove Diagnosis**

Write a query that shows both the REPORTED amount and the CORRECT amount side by side, proving the discrepancy. You'll need to insert the correct amounts manually (since the schema didn't capture them):

```sql
-- Show the gap: what the schema reports vs. what was actually paid
-- Hint: You know the historical prices from the INSERT comments above.
-- Create a comparison showing each enrollment with "reported" vs "actual" amounts.
```

---

**Task 2d: Fix the Schema**

Redesign the `enrollments` table so that the price at enrollment time is permanently captured. Write the new CREATE TABLE statement, migrate the existing data (using the known historical prices), and ensure future enrollments snapshot the price.

---

**Task 2e: Write a Verification Test**

Write a SQL script that:
1. Enrolls a student at the current price
2. Changes the course price
3. Queries the enrollment
4. Asserts that the enrollment still shows the ORIGINAL price

```sql
-- Your verification test here
-- It should demonstrate that price changes don't affect past enrollments
```

---

## Exercise 2.2: The Accidental Purge

**Context:** A CRM (Customer Relationship Management) system has been in production for 2 years. The company is cleaning up old data and deletes a company that went out of business. The next morning, the sales team discovers that 3 years of deal history, contact notes, and communication records have vanished.

```sql
CREATE TABLE companies (
    id          BIGSERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    industry    TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE contacts (
    id          BIGSERIAL PRIMARY KEY,
    company_id  BIGINT NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name        TEXT NOT NULL,
    email       TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE deals (
    id          BIGSERIAL PRIMARY KEY,
    contact_id  BIGINT NOT NULL REFERENCES contacts(id) ON DELETE CASCADE,
    title       TEXT NOT NULL,
    amount      NUMERIC(12,2) NOT NULL,
    stage       TEXT NOT NULL DEFAULT 'prospecting',
    closed_at   TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE activities (
    id          BIGSERIAL PRIMARY KEY,
    deal_id     BIGINT NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    type        TEXT NOT NULL CHECK (type IN ('email', 'call', 'meeting', 'note')),
    summary     TEXT NOT NULL,
    performed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed: 1 company, 2 contacts, 3 deals, 6 activities
INSERT INTO companies VALUES (1, 'Acme Corp', 'Manufacturing');
INSERT INTO contacts VALUES (1, 1, 'Jane Smith', 'jane@acme.com');
INSERT INTO contacts VALUES (2, 1, 'Tom Jones', 'tom@acme.com');
INSERT INTO deals VALUES (1, 1, 'Annual Contract', 50000, 'closed_won', '2024-06-01');
INSERT INTO deals VALUES (2, 1, 'Expansion Deal', 30000, 'negotiation', NULL);
INSERT INTO deals VALUES (3, 2, 'Consulting Gig', 15000, 'closed_won', '2024-09-15');
INSERT INTO activities VALUES (1, 1, 'email', 'Sent proposal', '2024-05-01');
INSERT INTO activities VALUES (2, 1, 'meeting', 'Contract review', '2024-05-20');
INSERT INTO activities VALUES (3, 1, 'note', 'Customer satisfied', '2024-07-01');
INSERT INTO activities VALUES (4, 2, 'call', 'Discussed expansion', '2025-01-10');
INSERT INTO activities VALUES (5, 3, 'email', 'Sent SOW', '2024-08-01');
INSERT INTO activities VALUES (6, 3, 'meeting', 'Kickoff call', '2024-09-20');
```

**The cleanup script runs:**

```sql
-- "Acme Corp went bankrupt. Remove them from our system."
DELETE FROM companies WHERE id = 1;
```

---

**Task 2a: Identify the Flaw**

1. Run the DELETE. Then run:
```sql
SELECT 'companies' AS tbl, COUNT(*) FROM companies
UNION ALL SELECT 'contacts', COUNT(*) FROM contacts
UNION ALL SELECT 'deals', COUNT(*) FROM deals
UNION ALL SELECT 'activities', COUNT(*) FROM activities;
```
2. How many total rows were destroyed?
3. Which foreign key declarations are responsible? Name each one and explain why its ON DELETE behavior is wrong.

---

**Task 2b: Explain the Mechanism**

1. Trace the cascade chain: which table triggers the deletion of which?
2. Why is CASCADE wrong for `contacts → companies`? What business value do contacts have independently?
3. Why is CASCADE wrong for `deals → contacts`? Consider: deals with `stage = 'closed_won'` represent revenue. What happens to the company's historical revenue records?
4. Is CASCADE appropriate for `activities → deals`? Justify your answer.

---

**Task 2c: Instrument to Prove Diagnosis**

Before running the DELETE (reset the data first), add a logging mechanism. Use a temporary table to capture what CASCADE would destroy:

```sql
-- Count what exists BEFORE the delete
SELECT 'BEFORE' AS timing, 'deals' AS tbl, COUNT(*) AS rows, SUM(amount) AS total_value 
FROM deals;

-- Run the DELETE
DELETE FROM companies WHERE id = 1;

-- Count what exists AFTER
SELECT 'AFTER' AS timing, 'deals' AS tbl, COUNT(*) AS rows, COALESCE(SUM(amount), 0) AS total_value 
FROM deals;

-- Show: $95,000 in deal value was destroyed by deleting ONE company row
```

---

**Task 2d: Fix the Schema**

Redesign the foreign key constraints with correct ON DELETE behavior:

1. Decide for each FK: should it be CASCADE, RESTRICT, or SET NULL? Justify each choice.
2. Add soft delete (`deleted_at`) where appropriate.
3. Write the corrected CREATE TABLE statements.

**Hint:** Not all CASCADE declarations are wrong. One of them is appropriate. Which one?

---

**Task 2e: Write a Verification Test**

Write a test that:
1. Attempts to delete a company that has contacts with deals
2. Verifies the DELETE is *blocked* (not silently cascaded)
3. Verifies that the deals and activities still exist after the failed delete

```sql
-- Your verification test here
-- Expected: DELETE fails with a foreign key violation
-- Expected: All deal and activity rows are preserved
```

---

# Level 2 Solutions

## Solution 2.1: The Price Chameleon

**Task 2a: Identify the Flaw**

After price updates, the revenue report shows:

```
 student | course         | enrollment_date | amount_paid
---------|----------------|-----------------|------------
 Alice   | Python Basics  | 2025-01-10      |       29.99   ← WRONG (was $49.99)
 Alice   | SQL Mastery    | 2025-01-15      |       59.99   ← WRONG (was $79.99)
 Bob     | Python Basics  | 2025-01-20      |       29.99   ← WRONG (was $49.99)
```

1. Alice's Python Basics shows $29.99. She paid $49.99. 
2. Reported total: $29.99 + $59.99 + $29.99 = $119.97. Actual total: $49.99 + $79.99 + $49.99 = $179.97.
3. Revenue is underreported by **$60.00** — a material accounting error.

**Precise flaw:** The `enrollments` table does not store the price at enrollment time. It relies on JOINing to the `courses.price` column, which stores only the *current* price.

**Task 2b: Explain the Mechanism**

1. The `enrollments` table has no price column. Every report must JOIN to `courses` to calculate amounts. The JOIN reads the live value in `courses.price`, which reflects the LATEST update — not the value at `enrolled_at` time.

2. The developer conflated two different facts: "the current price of a course" and "the amount a student paid at enrollment time." These happen to be equal at the moment of enrollment, but they diverge as soon as prices change. Normalization says don't store the *same fact* twice. These are *different facts* — one is a mutable property, the other is a historical record.

3. This is **Snapshot Data** from the lecture. The rule: if the source value CAN change and you need the HISTORICAL value, copy it at event time.

**Task 2c: Instrument to Prove Diagnosis**

```sql
SELECT 
    s.name,
    c.title,
    e.enrolled_at::DATE,
    c.price AS reported_price,
    CASE 
        WHEN c.id = 1 THEN 49.99   -- Known historical price for Python Basics
        WHEN c.id = 2 THEN 79.99   -- Known historical price for SQL Mastery
    END AS actual_price,
    c.price - CASE 
        WHEN c.id = 1 THEN 49.99 
        WHEN c.id = 2 THEN 79.99 
    END AS discrepancy
FROM enrollments e
JOIN students s ON s.id = e.student_id
JOIN courses c ON c.id = e.course_id
ORDER BY e.enrolled_at;
```

Output proves each enrollment has a nonzero discrepancy.

**Task 2d: Fix the Schema**

```sql
-- New enrollments table with price snapshot
CREATE TABLE enrollments_v2 (
    id           BIGSERIAL PRIMARY KEY,
    student_id   BIGINT NOT NULL REFERENCES students(id),
    course_id    BIGINT NOT NULL REFERENCES courses(id),
    amount_paid  NUMERIC(10,2) NOT NULL,   -- SNAPSHOT: frozen at enrollment time
    enrolled_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(student_id, course_id)
);

-- Migrate existing data (using known historical prices)
INSERT INTO enrollments_v2 (id, student_id, course_id, amount_paid, enrolled_at)
VALUES 
    (1, 1, 1, 49.99, '2025-01-10'),  -- Alice, Python Basics at original price
    (2, 1, 2, 79.99, '2025-01-15'),  -- Alice, SQL Mastery at original price
    (3, 2, 1, 49.99, '2025-01-20');  -- Bob, Python Basics at original price

-- Revenue report now uses the snapshot:
SELECT s.name, c.title, e.enrolled_at::DATE, e.amount_paid
FROM enrollments_v2 e
JOIN students s ON s.id = e.student_id
JOIN courses c ON c.id = e.course_id
ORDER BY e.enrolled_at;
-- Alice's Python Basics now correctly shows $49.99 regardless of current price.
```

**Task 2e: Verification Test**

```sql
-- Step 1: Record current price
-- Python Basics is currently $29.99 (after the sale)

-- Step 2: Enroll a new student at current price
INSERT INTO students VALUES (3, 'Dave', 'dave@mail.com', now());
INSERT INTO enrollments_v2 (student_id, course_id, amount_paid)
VALUES (3, 1, (SELECT price FROM courses WHERE id = 1));  
-- Dave enrolls at $29.99

-- Step 3: Price changes AGAIN
UPDATE courses SET price = 99.99 WHERE id = 1;

-- Step 4: Verify Dave's enrollment is frozen
SELECT amount_paid FROM enrollments_v2 WHERE student_id = 3 AND course_id = 1;
-- Expected: 29.99 (NOT 99.99)

-- If this returns 29.99, the fix works.
-- If it returned 99.99, the snapshot is broken.
```

---

## Solution 2.2: The Accidental Purge

**Task 2a: Identify the Flaw**

After the DELETE, all counts are 0. **12 total rows destroyed** (1 company + 2 contacts + 3 deals + 6 activities) from one DELETE statement.

The responsible FK declarations:
- `contacts.company_id REFERENCES companies(id) ON DELETE CASCADE` — Deleting a company destroys all its contacts.
- `deals.contact_id REFERENCES contacts(id) ON DELETE CASCADE` — Deleting contacts destroys all their deals.
- `activities.deal_id REFERENCES deals(id) ON DELETE CASCADE` — Deleting deals destroys all activity records.

**Task 2b: Explain the Mechanism**

1. Chain: `DELETE companies(1)` → CASCADE to `contacts(1,2)` → CASCADE to `deals(1,2,3)` → CASCADE to `activities(1-6)`.

2. CASCADE is wrong for contacts → companies because contacts have independent business value. Even after a company goes bankrupt, the contact persons may join other companies. Their contact information, communication history, and relationship context are valuable sales intelligence.

3. CASCADE is wrong for deals → contacts because deals with `stage = 'closed_won'` represent recognized revenue. Destroying them corrupts financial reporting, audit trails, and potentially violates legal record-retention requirements. $95,000 in deal value was silently erased.

4. CASCADE **is** appropriate for activities → deals. An activity log entry like "Sent proposal for Annual Contract" is meaningless without the deal it references. If a deal is (legitimately) deleted, its activity records serve no purpose. However, since deals themselves should NOT be cascaded, this CASCADE never triggers in the corrected design.

**Task 2d: Fix the Schema**

```sql
CREATE TABLE companies (
    id          BIGSERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    industry    TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ             -- Soft delete: preserve for history
);

CREATE TABLE contacts (
    id          BIGSERIAL PRIMARY KEY,
    company_id  BIGINT NOT NULL REFERENCES companies(id),  -- DEFAULT: RESTRICT
    name        TEXT NOT NULL,
    email       TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ             -- Soft delete
);

CREATE TABLE deals (
    id          BIGSERIAL PRIMARY KEY,
    contact_id  BIGINT NOT NULL REFERENCES contacts(id),   -- DEFAULT: RESTRICT
    title       TEXT NOT NULL,
    amount      NUMERIC(12,2) NOT NULL,
    stage       TEXT NOT NULL DEFAULT 'prospecting',
    closed_at   TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
    -- No deleted_at: deals are financial records. Never delete.
    -- Use stage = 'cancelled' instead.
);

CREATE TABLE activities (
    id           BIGSERIAL PRIMARY KEY,
    deal_id      BIGINT NOT NULL REFERENCES deals(id) ON DELETE CASCADE,  -- CASCADE: OK here
    type         TEXT NOT NULL CHECK (type IN ('email', 'call', 'meeting', 'note')),
    summary      TEXT NOT NULL,
    performed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Decision rationale:**
- `contacts → companies`: RESTRICT. Soft-delete the company first. Contacts can be reassigned or soft-deleted separately.
- `deals → contacts`: RESTRICT. Deals are financial records. A contact cannot be removed while deals reference it.
- `activities → deals`: CASCADE. Activity logs are subordinate to deals. In the corrected design, deals are never hard-deleted, so this CASCADE exists only as a safety-net.

**Task 2e: Verification Test**

```sql
-- Reset data (re-run the corrected CREATE TABLE + INSERT statements)

-- Test: Attempting to delete a company with contacts should FAIL
DELETE FROM companies WHERE id = 1;
-- Expected ERROR: update or delete on table "companies" violates foreign key 
--   constraint on table "contacts"

-- Verify: All data preserved
SELECT COUNT(*) FROM contacts;    -- Expected: 2
SELECT COUNT(*) FROM deals;       -- Expected: 3
SELECT SUM(amount) FROM deals;    -- Expected: 95000.00
SELECT COUNT(*) FROM activities;  -- Expected: 6

-- The correct workflow: soft-delete the company instead
UPDATE companies SET deleted_at = now() WHERE id = 1;

-- Company is hidden from active queries, but ALL data is preserved.
SELECT COUNT(*) FROM deals;       -- Still 3
SELECT SUM(amount) FROM deals;    -- Still 95000.00
```

---

# LEVEL 2.5: EXTEND EXISTING CODE

## Exercise 2.5: The Growing Blog Platform

**Background:** You've inherited a working blog platform schema. It's well-designed, all constraints are correct, and it has seed data. Your task is to extend it with three new features WITHOUT breaking existing behavior.

**Given Schema:**

```sql
-- ============================================
-- BLOG PLATFORM — WORKING SCHEMA
-- ============================================

CREATE TABLE authors (
    id         BIGSERIAL PRIMARY KEY,
    username   TEXT NOT NULL UNIQUE,
    email      TEXT NOT NULL UNIQUE,
    bio        TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE posts (
    id           BIGSERIAL PRIMARY KEY,
    author_id    BIGINT NOT NULL REFERENCES authors(id),
    title        TEXT NOT NULL,
    slug         TEXT NOT NULL UNIQUE,
    content      TEXT NOT NULL,
    published_at TIMESTAMPTZ,             -- NULL = draft, non-NULL = published
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE comments (
    id              BIGSERIAL PRIMARY KEY,
    post_id         BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    commenter_name  TEXT NOT NULL,
    commenter_email TEXT NOT NULL,
    content         TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed Data
INSERT INTO authors (id, username, email, bio) VALUES
    (1, 'alice', 'alice@blog.com', 'Backend developer'),
    (2, 'bob', 'bob@blog.com', 'Data engineer');

INSERT INTO posts (id, author_id, title, slug, content, published_at) VALUES
    (1, 1, 'Intro to SQL', 'intro-to-sql', 'SQL is great...', '2025-01-10'),
    (2, 1, 'Advanced JOINs', 'advanced-joins', 'Let us explore JOINs...', '2025-02-15'),
    (3, 2, 'Python Tips', 'python-tips', 'Here are some tips...', NULL);  -- Draft

INSERT INTO comments (id, post_id, commenter_name, commenter_email, content) VALUES
    (1, 1, 'Charlie', 'charlie@mail.com', 'Very helpful!'),
    (2, 1, 'Dana', 'dana@mail.com', 'Thanks for sharing.'),
    (3, 2, 'Charlie', 'charlie@mail.com', 'JOIN diagrams are useful.');
```

**Test Suite (MUST still pass after ALL extensions):**

```sql
-- TEST 1: Authors exist
SELECT COUNT(*) AS author_count FROM authors;
-- Expected: 2

-- TEST 2: Posts exist with correct authors
SELECT p.title, a.username 
FROM posts p JOIN authors a ON a.id = p.author_id 
ORDER BY p.id;
-- Expected: (Intro to SQL, alice), (Advanced JOINs, alice), (Python Tips, bob)

-- TEST 3: Comments cascade when post is deleted
DELETE FROM posts WHERE id = 2;
SELECT COUNT(*) FROM comments WHERE post_id = 2;
-- Expected: 0

-- TEST 4: Unique slug enforced
INSERT INTO posts (author_id, title, slug, content) 
VALUES (1, 'Duplicate', 'intro-to-sql', 'Test');
-- Expected: ERROR (unique violation)

-- TEST 5: Drafts have NULL published_at
SELECT title FROM posts WHERE published_at IS NULL;
-- Expected: Python Tips

-- (Re-insert deleted post for subsequent extensions)
INSERT INTO posts (id, author_id, title, slug, content, published_at) 
VALUES (2, 1, 'Advanced JOINs', 'advanced-joins', 'Let us explore JOINs...', '2025-02-15');
INSERT INTO comments (id, post_id, commenter_name, commenter_email, content) 
VALUES (3, 2, 'Charlie', 'charlie@mail.com', 'JOIN diagrams are useful.');
```

---

### Extension A: Tagging System (Many-to-Many)

**Requirement:** Posts can have tags (e.g., "python", "sql", "tutorial"). A post can have many tags. A tag can apply to many posts. Tags have a name that is globally unique (case-insensitive: "Python" and "python" are the same tag).

**Your Tasks:**

1. Create the `tags` table and the `post_tags` junction table.
2. Add appropriate primary keys, foreign keys, and constraints.
3. Decide: what ON DELETE behavior for each FK? Justify your choice.
4. Tag post 1 ("Intro to SQL") with "sql" and "tutorial". Tag post 2 ("Advanced JOINs") with "sql" and "advanced".

**Extension A Tests:**

```sql
-- TEST A1: Tags exist
SELECT name FROM tags ORDER BY name;
-- Expected: advanced, sql, tutorial

-- TEST A2: Post-tag associations work
SELECT p.title, t.name 
FROM post_tags pt
JOIN posts p ON p.id = pt.post_id
JOIN tags t ON t.id = pt.tag_id
WHERE p.id = 1
ORDER BY t.name;
-- Expected: (Intro to SQL, sql), (Intro to SQL, tutorial)

-- TEST A3: Duplicate tag-post association prevented
INSERT INTO post_tags (post_id, tag_id) VALUES (1, (SELECT id FROM tags WHERE name = 'sql'));
-- Expected: ERROR (unique or PK violation)

-- TEST A4: Deleting a post removes its tag associations (not the tags themselves)
DELETE FROM posts WHERE id = 2;
SELECT COUNT(*) FROM post_tags WHERE post_id = 2;
-- Expected: 0
SELECT COUNT(*) FROM tags;
-- Expected: 3 (tags still exist!)
```

---

### Extension B: Comment Threading (Self-Referential)

**Requirement:** Comments can now be replies to other comments. A comment may optionally have a `parent_id` pointing to another comment in the same post. Top-level comments have `parent_id = NULL`.

**Your Tasks:**

1. Add a `parent_id` column to the `comments` table.
2. What constraints do you need? Think about: FK reference, ON DELETE behavior, and preventing cross-post replies (a reply on post 1 cannot reference a comment on post 2).
3. Insert a reply from "Eve" to Charlie's comment (id=1) on "Intro to SQL".

**Extension B Tests:**

```sql
-- TEST B1: Top-level comments still work (parent_id = NULL)
SELECT COUNT(*) FROM comments WHERE parent_id IS NULL;
-- Expected: at least 2 (the original comments)

-- TEST B2: Reply exists with correct parent
SELECT c.content, c.parent_id, p.title
FROM comments c
JOIN posts p ON p.id = c.post_id
WHERE c.commenter_name = 'Eve';
-- Expected: (Eve's reply content, 1, Intro to SQL)

-- TEST B3: Deleting a parent comment handles replies
-- (What SHOULD happen? CASCADE the replies? SET NULL? RESTRICT? 
--  Justify your ON DELETE choice in a comment.)
```

---

### Extension C: Soft Delete for Posts

**Requirement:** Posts should no longer be hard-deleted. Instead, they should be soft-deleted (hidden but preserved). However, the unique slug constraint must still allow re-using a slug after soft-deleting its post.

**Your Tasks:**

1. Add a `deleted_at` column to the `posts` table.
2. Replace the standard UNIQUE constraint on `slug` with a partial unique index that only considers active (non-deleted) posts.
3. Modify the comments FK behavior: should comments of a soft-deleted post be preserved or deleted?
4. Update the seed data tests to use soft delete instead of hard delete.

**Extension C Tests:**

```sql
-- TEST C1: Soft-deleting a post preserves it in the table
UPDATE posts SET deleted_at = now() WHERE slug = 'intro-to-sql';
SELECT COUNT(*) FROM posts WHERE slug = 'intro-to-sql';
-- Expected: 1 (still exists!)

-- TEST C2: Slug can be reused after soft delete
INSERT INTO posts (author_id, title, slug, content, published_at)
VALUES (2, 'New Intro to SQL', 'intro-to-sql', 'Rewritten guide...', now());
-- Expected: SUCCESS (partial unique index allows this)

-- TEST C3: Two ACTIVE posts with the same slug are still prevented
INSERT INTO posts (author_id, title, slug, content, published_at)
VALUES (1, 'Yet Another SQL Intro', 'intro-to-sql', 'Another one...', now());
-- Expected: ERROR (partial unique index violation — slug already active)

-- TEST C4: Comments of soft-deleted post are preserved
SELECT COUNT(*) FROM comments c
JOIN posts p ON p.id = c.post_id
WHERE p.deleted_at IS NOT NULL;
-- Expected: > 0 (comments still linked to soft-deleted post)
```

---

# Level 2.5 Solutions

## Extension A Solution: Tagging System

```sql
CREATE TABLE tags (
    id   BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL UNIQUE   -- Globally unique tag names
);

CREATE TABLE post_tags (
    post_id BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    tag_id  BIGINT NOT NULL REFERENCES tags(id) ON DELETE RESTRICT,
    PRIMARY KEY (post_id, tag_id)  -- Composite PK: prevents duplicates and no surrogate ID needed
);
```

**ON DELETE justification:**
- `post_tags.post_id → posts ON DELETE CASCADE`: When a post is deleted, its tag associations should be removed. The tag-post link is meaningless without the post. But the tags themselves survive (the CASCADE is on the junction row, not the tag row).
- `post_tags.tag_id → tags ON DELETE RESTRICT`: Prevent deleting a tag while posts use it. Remove tag associations first, then delete the tag.

**Seed the tag data:**
```sql
INSERT INTO tags (id, name) VALUES (1, 'sql'), (2, 'tutorial'), (3, 'advanced');
INSERT INTO post_tags VALUES (1, 1), (1, 2);  -- Post 1: sql, tutorial
INSERT INTO post_tags VALUES (2, 1), (2, 3);  -- Post 2: sql, advanced
```

**Why composite PK?** This is a pure junction table — no other table references it. The composite PK `(post_id, tag_id)` both enforces uniqueness and serves as the primary key. No surrogate ID needed (lecture Part 3.4).

---

## Extension B Solution: Comment Threading

```sql
ALTER TABLE comments 
ADD COLUMN parent_id BIGINT REFERENCES comments(id) ON DELETE CASCADE;
```

**ON DELETE CASCADE justification:** If a parent comment is deleted, its replies become orphaned and contextually meaningless ("I agree!" — with what?). CASCADE removes the reply chain. An alternative is SET NULL (replies become top-level comments), which is also valid depending on UX preferences. The key is making a deliberate choice.

**Note on cross-post prevention:** A pure SQL constraint cannot easily prevent a reply on post 1 from referencing a comment on post 2 (this requires a trigger or application-level validation). A comment documenting this limitation is appropriate:

```sql
-- WARNING: No DB-level constraint prevents cross-post replies.
-- Application code must validate that parent comment belongs to the same post.
-- A trigger could enforce this, but is beyond scope for this exercise.
```

**Insert the reply:**
```sql
INSERT INTO comments (post_id, commenter_name, commenter_email, content, parent_id)
VALUES (1, 'Eve', 'eve@mail.com', 'Great point Charlie! I had the same experience.', 1);
```

---

## Extension C Solution: Soft Delete for Posts

```sql
-- Step 1: Add deleted_at column
ALTER TABLE posts ADD COLUMN deleted_at TIMESTAMPTZ;

-- Step 2: Drop the old UNIQUE constraint on slug
ALTER TABLE posts DROP CONSTRAINT posts_slug_key;

-- Step 3: Create partial unique index (only active posts)
CREATE UNIQUE INDEX idx_posts_active_slug
ON posts (slug)
WHERE deleted_at IS NULL;

-- Step 4: Comments FK — KEEP CASCADE? Or change?
-- Decision: Keep ON DELETE CASCADE on comments.post_id.
-- But since we no longer hard-delete posts, CASCADE never triggers.
-- Soft-deleted posts retain their comments (for potential restore).
-- If we later add hard-delete-after-retention-period (background job),
-- CASCADE will clean up comments at that point.
```

**Why keep CASCADE?** With soft deletes, we never actually DELETE the post row, so CASCADE is irrelevant during normal operation. If a background job eventually purges old soft-deleted posts (GDPR compliance, storage cleanup), CASCADE automatically cleans up their comments. Belt AND suspenders.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## Exercise 3: EventHub — Event Ticketing Platform

**Scenario:** You are the backend engineer for EventHub. Your PM provides the following requirements. Design a schema from scratch, enforcing every business rule at the database level.

**Business Requirements:**

```
EventHub is an event ticketing platform.

 • Organizers create events. An organizer is a user with role = 'organizer'.
 • Each event has a name, description, date/time, and a venue (just a text 
   field for v1 — not a separate table).
 • Each event has one or more ticket tiers (e.g., "VIP", "General Admission").
 • Each tier has a price and a maximum capacity (number of tickets available).
 • Users purchase tickets. A single purchase is for ONE event only.
 • Each purchase contains one or more line items (a tier + quantity).
 • The price must be captured at purchase time (tier prices may change).
 • A user cannot purchase more than 4 tickets total per event 
   (across all tiers combined).
 • Users can review an event: 1-5 stars + optional comment.
 • A user can only review an event ONCE.
 • A user can only review an event whose date has PASSED 
   (no reviewing future events).
 • Events can be cancelled. Cancelled events must NOT delete purchase 
   records (needed for refund processing).
 • Track created_at and updated_at on all major tables.
```

---

### Task 3a: Design the Core Schema

Design the complete schema. For each table, specify:
- All columns with data types
- Primary keys (choose UUID vs BIGSERIAL deliberately)
- Foreign keys with ON DELETE behavior
- NOT NULL, UNIQUE, CHECK constraints
- Timestamps (created_at, updated_at, deleted_at where appropriate)

**Test Suite for 3a:**

```sql
-- TEST 3a-1: Can create an organizer and event
INSERT INTO users (id, name, email, role) 
VALUES (1, 'Organizer Pat', 'pat@events.com', 'organizer');
INSERT INTO events (id, organizer_id, name, venue, starts_at)
VALUES (1, 1, 'Tech Conference', 'Grand Hall', '2025-09-15 09:00:00+00');
-- Expected: SUCCESS

-- TEST 3a-2: Can create ticket tiers
INSERT INTO ticket_tiers (id, event_id, name, price, capacity)
VALUES (1, 1, 'General Admission', 49.99, 200),
       (2, 1, 'VIP', 149.99, 50);
-- Expected: SUCCESS

-- TEST 3a-3: Tier prices must be positive
INSERT INTO ticket_tiers (id, event_id, name, price, capacity)
VALUES (3, 1, 'Free Tier', 0, 100);
-- Expected: SUCCESS (0 is valid for free events)
INSERT INTO ticket_tiers (id, event_id, name, price, capacity)
VALUES (4, 1, 'Negative', -10, 100);
-- Expected: ERROR (CHECK violation)

-- TEST 3a-4: Capacity must be positive
INSERT INTO ticket_tiers (id, event_id, name, price, capacity)
VALUES (5, 1, 'Empty', 10, 0);
-- Expected: ERROR (CHECK violation: capacity must be > 0)

-- TEST 3a-5: Can create a purchase with line items (price snapshot)
INSERT INTO users (id, name, email, role) VALUES (2, 'Alice', 'alice@mail.com', 'attendee');
INSERT INTO purchases (id, user_id, event_id) VALUES (1, 2, 1);
INSERT INTO purchase_items (id, purchase_id, tier_id, quantity, unit_price)
VALUES (1, 1, 1, 2, 49.99);
-- Expected: SUCCESS

-- TEST 3a-6: Cancelling an event preserves purchase records
UPDATE events SET status = 'cancelled', cancelled_at = now() WHERE id = 1;
SELECT COUNT(*) FROM purchases WHERE event_id = 1;
-- Expected: 1 (purchase preserved!)
SELECT COUNT(*) FROM purchase_items WHERE purchase_id = 1;
-- Expected: 1 (line items preserved!)
```

---

### Task 3b: Enforce Business Rules at the Database Level

Add constraints for these three rules. Some may require CHECK constraints, UNIQUE constraints, or triggers.

**Rule 1: One review per user per event.**

**Rule 2: Review rating must be between 1 and 5.**

**Rule 3: A user can only review an event whose `starts_at` is in the past.**
(Hint: this rule is difficult or impossible to enforce with a pure CHECK constraint. Explain why, and propose an alternative. If you use a trigger, write it. If you argue it should be application-level, justify why.)

**Test Suite for 3b:**

```sql
-- Set event to past date for testing
UPDATE events SET starts_at = '2025-01-15 09:00:00+00' WHERE id = 1;

-- TEST 3b-1: Valid review succeeds
INSERT INTO reviews (user_id, event_id, rating, comment)
VALUES (2, 1, 4, 'Great conference!');
-- Expected: SUCCESS

-- TEST 3b-2: Duplicate review blocked
INSERT INTO reviews (user_id, event_id, rating, comment)
VALUES (2, 1, 5, 'Changed my mind, 5 stars!');
-- Expected: ERROR (unique violation)

-- TEST 3b-3: Invalid rating blocked
INSERT INTO users (id, name, email, role) VALUES (3, 'Bob', 'bob@mail.com', 'attendee');
INSERT INTO reviews (user_id, event_id, rating) VALUES (3, 1, 6);
-- Expected: ERROR (CHECK violation)

-- TEST 3b-4: Review of future event blocked
INSERT INTO events (id, organizer_id, name, venue, starts_at)
VALUES (2, 1, 'Future Fest', 'Arena', '2027-12-01 09:00:00+00');
INSERT INTO reviews (user_id, event_id, rating) VALUES (3, 2, 5);
-- Expected: ERROR (event hasn't happened yet)
```

---

### Task 3c: Add an Audit Trail

**Requirement:** Track every status change for events and purchases. When an event moves from 'upcoming' to 'cancelled', or a purchase moves from 'confirmed' to 'refunded', a row should be inserted in an audit table recording: who changed it, when, from what status, to what status.

**Your Tasks:**
1. Design the audit table(s).
2. Write a trigger that automatically logs status changes for the `events` table.
3. Populate it by changing an event's status.

**Test Suite for 3c:**

```sql
-- TEST 3c-1: Status change creates audit record
UPDATE events SET status = 'cancelled', cancelled_at = now() WHERE id = 1;
SELECT old_status, new_status, changed_at
FROM event_status_audit
WHERE event_id = 1
ORDER BY changed_at DESC
LIMIT 1;
-- Expected: ('upcoming', 'cancelled', <recent timestamp>)

-- TEST 3c-2: Multiple status changes create multiple audit records
-- (Reset event for testing)
UPDATE events SET status = 'upcoming', cancelled_at = NULL WHERE id = 1;
UPDATE events SET status = 'cancelled', cancelled_at = now() WHERE id = 1;
SELECT COUNT(*) FROM event_status_audit WHERE event_id = 1;
-- Expected: at least 3 (initial → cancelled → upcoming → cancelled)
```

---

# Level 3 Solutions

## Solution 3a: Core Schema

```sql
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    email      TEXT NOT NULL UNIQUE,
    role       TEXT NOT NULL DEFAULT 'attendee'
               CHECK (role IN ('attendee', 'organizer', 'admin')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE events (
    id            BIGSERIAL PRIMARY KEY,
    organizer_id  BIGINT NOT NULL REFERENCES users(id),
    name          TEXT NOT NULL,
    description   TEXT,
    venue         TEXT NOT NULL,
    starts_at     TIMESTAMPTZ NOT NULL,
    status        TEXT NOT NULL DEFAULT 'upcoming'
                  CHECK (status IN ('upcoming', 'cancelled', 'completed')),
    cancelled_at  TIMESTAMPTZ,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
    -- No deleted_at: use status = 'cancelled' instead.
    -- Purchase records must survive, so never hard-delete events.
);

CREATE TABLE ticket_tiers (
    id         BIGSERIAL PRIMARY KEY,
    event_id   BIGINT NOT NULL REFERENCES events(id),  -- RESTRICT: don't delete event if tiers exist
    name       TEXT NOT NULL,
    price      NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    capacity   INTEGER NOT NULL CHECK (capacity > 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(event_id, name)  -- No two tiers with same name per event
);

CREATE TABLE purchases (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT NOT NULL REFERENCES users(id),
    event_id   BIGINT NOT NULL REFERENCES events(id),  -- RESTRICT: preserve purchases
    status     TEXT NOT NULL DEFAULT 'confirmed'
               CHECK (status IN ('confirmed', 'refunded', 'cancelled')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE purchase_items (
    id           BIGSERIAL PRIMARY KEY,
    purchase_id  BIGINT NOT NULL REFERENCES purchases(id) ON DELETE CASCADE,
    tier_id      BIGINT NOT NULL REFERENCES ticket_tiers(id),
    quantity     INTEGER NOT NULL CHECK (quantity > 0),
    unit_price   NUMERIC(10,2) NOT NULL,  -- SNAPSHOT: price at purchase time
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
    -- No updated_at: purchase items are immutable after creation.
);

CREATE TABLE reviews (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT NOT NULL REFERENCES users(id),
    event_id   BIGINT NOT NULL REFERENCES events(id),
    rating     INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment    TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, event_id)  -- One review per user per event
);
```

**Design decisions:**
- BIGSERIAL PKs throughout (this is an internal system; if it were API-facing, UUIDs for users, events, purchases).
- Events use `status` column instead of `deleted_at` because cancellation is a business state, not a deletion.
- `purchase_items ON DELETE CASCADE` from purchases: line items are meaningless without the purchase. But purchases themselves use RESTRICT from events — they must survive event cancellation.
- `unit_price` on purchase_items: snapshot pattern. Tier price changes don't corrupt past purchases.

---

## Solution 3b: Business Rules

**Rule 1 & 2:** Already handled by `UNIQUE(user_id, event_id)` and `CHECK (rating BETWEEN 1 AND 5)` on the reviews table.

**Rule 3 (review only past events):** A CHECK constraint on the `reviews` table cannot reference another table (`events`). PostgreSQL CHECK constraints can only reference columns of the same row. This is a cross-table constraint and requires a trigger.

```sql
CREATE OR REPLACE FUNCTION check_event_has_passed()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT starts_at FROM events WHERE id = NEW.event_id) > now() THEN
        RAISE EXCEPTION 'Cannot review an event that has not yet occurred.';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_review_event_must_be_past
    BEFORE INSERT ON reviews
    FOR EACH ROW
    EXECUTE FUNCTION check_event_has_passed();
```

**Why not a CHECK constraint?** PostgreSQL's CHECK constraints are evaluated per-row and cannot contain subqueries or references to other tables. The constraint `CHECK (event_starts_at < now())` would require `event_starts_at` to be a column on the reviews table (which would be denormalization). A trigger is the correct tool for cross-table invariants.

---

## Solution 3c: Audit Trail

```sql
CREATE TABLE event_status_audit (
    id          BIGSERIAL PRIMARY KEY,
    event_id    BIGINT NOT NULL REFERENCES events(id),
    old_status  TEXT NOT NULL,
    new_status  TEXT NOT NULL,
    changed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    changed_by  TEXT  -- Could be a user_id FK; TEXT for simplicity in v1
);

CREATE OR REPLACE FUNCTION log_event_status_change()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.status IS DISTINCT FROM NEW.status THEN
        INSERT INTO event_status_audit (event_id, old_status, new_status)
        VALUES (OLD.id, OLD.status, NEW.status);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_event_status
    AFTER UPDATE ON events
    FOR EACH ROW
    EXECUTE FUNCTION log_event_status_change();
```

**Why `IS DISTINCT FROM` instead of `!=`?** If `OLD.status` or `NEW.status` were NULL, the `!=` comparison returns UNKNOWN and the IF block would not execute. `IS DISTINCT FROM` treats NULL as a regular value (NULL IS DISTINCT FROM 'upcoming' = TRUE). This is defensive coding at the SQL level.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## Exercise 4: When the "Best Practice" Backfires

**Task 4a:** Describe a concrete scenario where **soft delete** creates a *worse* outcome than hard delete. Write SQL demonstrating the failure mode. Your scenario must include:
- A schema with soft deletes
- A sequence of operations that leads to a problem
- An explanation of why hard delete would have been better

**Task 4b:** Describe a concrete scenario where using **UUID primary keys** causes a measurable problem compared to BIGSERIAL. Your answer must go beyond "UUIDs are larger" — show a specific operational consequence.

---

# Level 4 Solutions

## Solution 4a: Soft Delete — The GDPR Nightmare

**Scenario:** A European user invokes their GDPR "right to erasure" — they want ALL their personal data deleted from the system. The platform uses soft deletes.

```sql
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    email      TEXT NOT NULL UNIQUE,
    name       TEXT NOT NULL,
    phone      TEXT,
    deleted_at TIMESTAMPTZ
);

CREATE TABLE user_addresses (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT NOT NULL REFERENCES users(id),
    street     TEXT NOT NULL,
    city       TEXT NOT NULL,
    postal     TEXT NOT NULL,
    deleted_at TIMESTAMPTZ
);

INSERT INTO users VALUES (1, 'hans@example.de', 'Hans Mueller', '+49-555-1234', NULL);
INSERT INTO user_addresses VALUES (1, 1, 'Berliner Str. 42', 'Berlin', '10115', NULL);

-- "Delete" the user (soft delete)
UPDATE users SET deleted_at = now(), email = 'deleted', name = 'deleted', phone = NULL WHERE id = 1;
UPDATE user_addresses SET deleted_at = now() WHERE user_id = 1;

-- Compliance officer checks: "Is Hans's data actually gone?"
SELECT * FROM users WHERE id = 1;
-- Hans's row STILL EXISTS. Even with name='deleted', the ID, created_at, 
-- and the row's existence can be correlated with other systems.
-- The address row still has street, city, postal code.
```

**Why hard delete is better here:** GDPR's "right to erasure" means the data must actually be removed, not hidden. Soft delete preserves the data. A compliant approach for this table is hard delete (or anonymize + hard-delete the address). Soft delete is wrong when legal requirements demand actual data removal.

**The Key Insight:** Soft delete is not universally "safer." For privacy-sensitive data subject to deletion laws, soft delete can be a legal liability. The right approach depends on whether the data has retention requirements (keep it) or deletion requirements (destroy it).

## Solution 4b: UUID — The Index Fragmentation Problem

**Scenario:** A high-write table uses UUIDv4 (random) as its primary key. At scale, insert performance degrades.

```sql
CREATE TABLE sensor_readings_uuid (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id  INTEGER NOT NULL,
    value      NUMERIC NOT NULL,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sensor_readings_serial (
    id         BIGSERIAL PRIMARY KEY,
    sensor_id  INTEGER NOT NULL,
    value      NUMERIC NOT NULL,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

With BIGSERIAL, every new row's ID is greater than the last. The B-tree index appends new entries to the rightmost leaf page — this is fast and cache-friendly.

With UUIDv4, every new ID is random. The B-tree index must insert into a random leaf page. At high volume (millions of rows), this causes: random I/O (disk seeks instead of sequential writes), B-tree page splits across the entire index (not just the rightmost page), and much larger working set in memory (the entire index is "hot," not just the tail).

For an IoT platform ingesting thousands of readings per second, this difference can mean 2-5x slower inserts. The solution is either BIGSERIAL (best insert performance) or UUIDv7 (time-sortable, so inserts are approximately sequential).

**The Key Insight:** UUID is not "always safer." For internal, high-write tables that are never exposed in APIs, BIGSERIAL is faster and more space-efficient. Reserve UUID for tables whose IDs appear in public URLs or cross-system references.

---

# LEVEL 5: THE TEACH BACK

## Exercise 5

**Scenario:** A junior developer on your team reviews a pull request that adds foreign key constraints to the database schema. They leave this comment:

> "I don't understand why we need all these REFERENCES clauses. My application code already checks that the restaurant_id exists before inserting a menu item. I validated this in my tests — the INSERT only runs if the restaurant exists. Adding REFERENCES is just extra complexity for no benefit. Plus, ON DELETE RESTRICT is annoying — I keep getting errors when I try to clean up test data. Can we just remove all the foreign key constraints and handle this in application code?"

**Your Task:**

Write a response (as if replying to the PR comment) that:

1. Acknowledges what the developer got right (their application validation IS doing something useful).
2. Explains the specific failure modes that occur WITHOUT database constraints, even when application validation exists.
3. Gives at least two concrete examples of how data corruption happens despite "correct" application code.
4. Addresses their complaint about RESTRICT being "annoying" during testing.
5. Does NOT be condescending. This is a teaching moment, not a lecture.

---

# Level 5 Solution

**Response to the PR comment:**

You're right that the application checks work — and testing them was the right call. But application-level validation and database constraints solve different problems, and we need both. Here's why.

**What your application validation cannot handle:**

**1. Race conditions.** Two requests arrive simultaneously. Both check "does restaurant 5 exist?" — both get YES. Then restaurant 5 is deleted by a third request between the checks and the inserts. Both INSERTs proceed, creating menu items pointing to a restaurant that no longer exists. This isn't theoretical — it's a standard concurrency bug called a TOCTOU (time-of-check to time-of-use) race. Foreign key constraints are evaluated atomically within the INSERT transaction, eliminating this window entirely.

**2. Multiple entry points.** Right now, your code validates before INSERT. But six months from now, someone writes a data migration script, an admin CLI tool, or a bulk import function. They don't know about your validation function. They INSERT directly. Without REFERENCES, the database accepts garbage data silently. With REFERENCES, the database rejects invalid data regardless of which code path triggered the INSERT — your API handler, a migration script, a psql session, or a future service that doesn't exist yet.

**3. Bug immunity.** Your validation code can have bugs. If someone accidentally comments out the check during a refactor, or if the check has an off-by-one error in the query, invalid data flows in. The database constraint is immune to application bugs — it's enforced by PostgreSQL's storage engine, not by your code. It's the safety net that catches what your code misses.

**On RESTRICT being annoying during testing:** That frustration is real and valid. The fix isn't removing constraints — it's structuring your test cleanup to respect the dependency order. Delete children before parents: `DELETE FROM menu_items WHERE restaurant_id = 5; DELETE FROM restaurants WHERE id = 5;`. Or use a test fixture that truncates tables in reverse-dependency order: `TRUNCATE menu_items, restaurants CASCADE;`. The TRUNCATE...CASCADE is safe in test databases where you want a clean slate. The RESTRICT errors you're hitting in development are exactly the errors that would protect production data from accidental deletion. The friction in dev is the feature in prod.