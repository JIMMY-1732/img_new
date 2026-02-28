# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "SQL is basically Python string filtering with different syntax. `WHERE` is just an `if` statement, `JOIN` just sticks columns together, and `NULL` is Python's `None`."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | `NULL` works like Python's `None` — you can compare it with `=` and `!=` | `NULL` means "unknown." ANY comparison with `NULL` produces `NULL`, not `TRUE` or `FALSE`. `NULL != 5` is `NULL`, not `TRUE`. |
| 2 | `SELECT` runs first because you write it first | SQL execution order is: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT. SELECT runs near the end. |
| 3 | `JOIN` adds columns to your table without changing the number of rows | A JOIN can **multiply** rows. If Alice has 3 assignments, she appears 3 times after joining with assignments. |
| 4 | `LEFT JOIN` always preserves all left-table rows | A `WHERE` clause on a right-table column **silently converts** a LEFT JOIN into an INNER JOIN by filtering out `NULL` rows. |
| 5 | `COUNT(*)` and `COUNT(column)` are the same thing | `COUNT(*)` counts all rows including NULLs. `COUNT(column)` skips `NULL` values. With LEFT JOINs, this difference is critical. |
| 6 | `NOT IN` is the negation of `IN` and always works symmetrically | If the subquery list contains even one `NULL`, `NOT IN` returns **zero rows** for every value — silently. |
| 7 | `SUM` and `AVG` over a joined table give correct totals | If a JOIN duplicates rows, `SUM` and `AVG` produce inflated numbers. The query runs, returns results, and nothing signals the numbers are wrong. |

**The Prediction Gap:** When students see `WHERE department_id != 2`, they predict it returns every employee not in department 2 — including Grace Kim (whose `department_id` is `NULL`). The actual behavior is that Grace is silently excluded because `NULL != 2` evaluates to `NULL`, not `TRUE`. The query returns fewer rows than expected with no error and no warning.

---

## The Main Purpose (Essence)

**Core Insight:** SQL is a **declarative set-operation language**, not an imperative row-processing loop. You describe the shape of the result set you want. The database engine decides how to produce it. Every SQL concept — `WHERE`, `JOIN`, `GROUP BY`, `HAVING` — is a **set transformation** that reshapes, combines, or collapses sets of rows.

**The Problems Being Solved:**

| Concept | What Students Think It Does | What It Actually Does |
|---------|---------------------------|----------------------|
| `WHERE` | "Loops through rows and checks each one" | Defines a **predicate** that filters the row set. Rows where the predicate is not `TRUE` (including `NULL`) are removed. |
| `JOIN` | "Sticks columns from two tables side by side" | Computes the **cross product** of two tables, then filters by the `ON` condition. Can produce fewer rows (INNER) or more rows (one-to-many) than either input. |
| `GROUP BY` | "Sorts rows into buckets" | **Collapses** each group into a single output row. Non-aggregated columns cannot appear in SELECT because they represent multiple values collapsed into one. |
| `HAVING` | "Same as WHERE but for groups" | Filters **after** aggregation. It operates on collapsed groups, not individual rows. This is why it can reference aggregate functions but WHERE cannot. |

**The Key Distinction:**
- **Imperative (Python):** You describe the *steps*: loop, check, append, sort.
- **Declarative (SQL):** You describe the *result*: "rows where X is true, grouped by Y, sorted by Z."
- The mental shift is from "how do I process each row?" to "what does the result set look like?"

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "NULL is a value like 0 or empty string" | `WHERE dept_id != 2` includes Grace (NULL) | Grace excluded — 4 rows, not 5 | `NULL != 2` → `NULL`, not `TRUE`. WHERE keeps only `TRUE`. |
| "LEFT JOIN always keeps all left rows" | LEFT JOIN + `WHERE d.name != 'X'` keeps unmatched rows | Unmatched rows filtered out — LEFT JOIN becomes INNER JOIN | Right-table columns are `NULL` for unmatched rows. `NULL != 'X'` → `NULL`. |
| "JOIN just adds columns" | `SUM(salary)` after JOIN gives correct total | Salary summed multiple times per employee | JOIN on assignments duplicates employee rows per project. |
| "NOT IN is the opposite of IN" | `WHERE id NOT IN (subquery)` finds unmatched rows | Returns 0 rows | If subquery contains `NULL`, every `NOT IN` comparison becomes `NULL`. |
| "COUNT(*) counts the things I care about" | LEFT JOIN + `COUNT(*)` gives 0 for empty groups | Returns 1 for empty groups | `COUNT(*)` counts the row itself (which exists from LEFT JOIN). |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The NULL Exclusion

```sql
-- "Show me all employees who are NOT in Marketing (department_id = 2)"
SELECT name, department_id
FROM employees
WHERE department_id != 2;
```

**Questions:**

1. **Predict the result.** How many rows does this return? List every employee name you expect to see.

2. **Explain the gap.** A student says: "Grace Kim's `department_id` is NULL, which is definitely not 2, so she should appear." Why is this reasoning wrong? What does `NULL != 2` actually evaluate to?

3. **Mental Trace:** Walk through the WHERE clause evaluation for **each** of the 7 employees. For each row, write the expression `department_id != 2`, its result (`TRUE`, `FALSE`, or `NULL`), and whether the row passes the filter.

---

## Exercise 1.2: The LEFT JOIN Eraser

A developer wants to list all employees with their department names, **excluding anyone in Marketing**. They use LEFT JOIN because they want to keep Grace Kim (who has no department):

```sql
SELECT e.name, d.name AS department
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
WHERE d.name != 'Marketing';
```

**Questions:**

1. **Predict the result.** How many rows? Which employees appear? Specifically: does Grace Kim appear?

2. **Explain the mechanism.** The developer chose LEFT JOIN specifically to keep Grace. Why did it fail? At what point in the SQL execution pipeline does Grace's row get removed?

3. **Two fixes exist.** One keeps the WHERE clause, the other moves the condition to the ON clause. Write both. Do they produce the same result? If not, what is different and why?

---

## Exercise 1.3: The Multiplied Payroll

The finance team asks for a report showing each department's headcount and total payroll, along with how many project assignments exist in each department:

```sql
SELECT
    d.name AS department,
    COUNT(*) AS headcount,
    SUM(e.salary) AS total_payroll,
    COUNT(a.project_id) AS total_assignments
FROM departments d
INNER JOIN employees e ON d.id = e.department_id
INNER JOIN assignments a ON e.id = a.employee_id
GROUP BY d.id, d.name
ORDER BY total_payroll DESC;
```

**Questions:**

1. **Predict the result.** For Engineering specifically: what headcount, total payroll, and total assignments do you expect? (Engineering has Alice at 95k, Bob at 85k, David at 110k.)

2. **Trace the JOIN multiplication.** Before GROUP BY runs, how many rows exist for Engineering after both JOINs? List each row (employee name + project name). Then explain why `SUM(salary)` and `COUNT(*)` produce inflated numbers.

3. **Explain the discrepancy.** Marketing shows `headcount = 2` and `total_payroll = 145000`, which happen to be correct. Why does Marketing escape the inflation bug while Engineering does not? What property of the data causes this difference?

---

## Exercise 1.4: The NOT IN Black Hole

A developer writes a query to find departments that have **no employees assigned**:

```sql
SELECT name
FROM departments
WHERE id NOT IN (
    SELECT department_id
    FROM employees
);
```

**Questions:**

1. **Predict the result.** Which departments do you expect to appear? (Hint: look at the `employees` table — which `department_id` values exist?)

2. **Run the query.** How many rows are returned? Were you correct?

3. **Explain the black hole.** The subquery `SELECT department_id FROM employees` returns the values: `1, 1, 2, 1, 3, 2, NULL`. Walk through the evaluation of `4 NOT IN (1, 1, 2, 1, 3, 2, NULL)` step by step. What does each comparison produce? What does the final combined expression evaluate to? Why does this prevent Sales from appearing?

---

# Level 1 Solutions

## Solution 1.1: The NULL Exclusion

**1. Predicted vs. Actual Result:**

```
    name     | department_id
─────────────┼───────────────
 Alice Chen  │             1
 Bob Smith   │             1
 David Lee   │             1
 Eve Wilson  │             3
(4 rows)
```

**4 rows, not 5.** Grace Kim is missing.

**2. Why the reasoning is wrong:**

`NULL` is not a value — it means "unknown." The expression `NULL != 2` does not evaluate to `TRUE`. It evaluates to `NULL`. And `WHERE` only keeps rows where the condition is `TRUE`. The row-by-row evaluation:

| Employee | `department_id != 2` | Result | Kept? |
|---|---|---|---|
| Alice Chen | `1 != 2` | `TRUE` | ✅ |
| Bob Smith | `1 != 2` | `TRUE` | ✅ |
| Carol Jones | `2 != 2` | `FALSE` | ❌ |
| David Lee | `1 != 2` | `TRUE` | ✅ |
| Eve Wilson | `3 != 2` | `TRUE` | ✅ |
| Frank Brown | `2 != 2` | `FALSE` | ❌ |
| Grace Kim | `NULL != 2` | **`NULL`** | ❌ |

`WHERE` has a three-valued logic: `TRUE` → keep, `FALSE` → discard, `NULL` → **discard**. Grace is discarded not because she's in department 2, but because the database cannot determine whether her unknown department is or isn't 2.

**3. The Fix:**

```sql
SELECT name, department_id
FROM employees
WHERE department_id != 2 OR department_id IS NULL;
```

**The Key Insight:** Any comparison involving `NULL` produces `NULL`, not a boolean. This means `NULL` is invisible to both `=` and `!=`. You must **always** handle `NULL` explicitly with `IS NULL` / `IS NOT NULL` when your data contains nullable columns.

---

## Solution 1.2: The LEFT JOIN Eraser

**1. Actual Result:**

```
    name     |  department
─────────────┼─────────────
 Alice Chen  │ Engineering
 Bob Smith   │ Engineering
 David Lee   │ Engineering
 Eve Wilson  │ Design
(4 rows)
```

**4 rows.** Grace Kim is excluded. The LEFT JOIN was effectively converted into an INNER JOIN.

**2. The mechanism:**

After the LEFT JOIN executes, the intermediate result is:

| e.name | d.name |
|---|---|
| Alice Chen | Engineering |
| Bob Smith | Engineering |
| Carol Jones | Marketing |
| David Lee | Engineering |
| Eve Wilson | Design |
| Frank Brown | Marketing |
| Grace Kim | **NULL** |

Now `WHERE d.name != 'Marketing'` evaluates per row:

| Row | `d.name != 'Marketing'` | Result | Kept? |
|---|---|---|---|
| Alice | `'Engineering' != 'Marketing'` | `TRUE` | ✅ |
| Carol | `'Marketing' != 'Marketing'` | `FALSE` | ❌ |
| Grace | `NULL != 'Marketing'` | **`NULL`** | ❌ |

Grace's `d.name` is `NULL` because the LEFT JOIN found no matching department. `NULL != 'Marketing'` evaluates to `NULL`. The WHERE clause removes it. The LEFT JOIN's preservation guarantee is **undone** by the WHERE filter.

Recall the execution order: FROM (with JOIN) → WHERE. The LEFT JOIN happens first and correctly keeps Grace with NULLs. But then WHERE runs and throws her out.

**3. Two Fixes:**

**Fix A — Keep the WHERE, add NULL guard:**

```sql
SELECT e.name, d.name AS department
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
WHERE d.name != 'Marketing' OR d.name IS NULL;
```

Result: Alice, Bob, David, Eve, **Grace** (5 rows). Grace appears with NULL department.

**Fix B — Move the condition into the ON clause:**

```sql
SELECT e.name, d.name AS department
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id AND d.name != 'Marketing';
```

Result: All 7 employees. But Carol and Frank now show `NULL` for department instead of being excluded entirely. The condition in `ON` controls **which departments match**, not **which employees appear**. Marketing employees remain but lose their department label.

**The Key Insight:** Fix A and Fix B produce different results because they answer different questions. Fix A asks "exclude Marketing employees, keep unassigned ones." Fix B asks "show all employees, but don't reveal the Marketing department name." The placement of the condition — `ON` vs `WHERE` — changes the semantics of the query.

---

## Solution 1.3: The Multiplied Payroll

**1. Actual Result:**

```
  department  | headcount | total_payroll | total_assignments
──────────────┼───────────┼───────────────┼───────────────────
 Engineering  │         7 │        675000 │                 7
 Design       │         2 │        160000 │                 2
 Marketing    │         2 │        145000 │                 2
(3 rows)
```

Engineering shows headcount **7** and payroll **675,000**. The correct values are headcount 3 and payroll 290,000. Sales is missing entirely (INNER JOIN drops it).

**2. The JOIN multiplication trace:**

After `departments JOIN employees JOIN assignments`, the rows for Engineering are:

| e.name | e.salary | p.name (via a.project_id) |
|---|---|---|
| Alice Chen | 95,000 | API Redesign |
| Alice Chen | 95,000 | Mobile App |
| Alice Chen | 95,000 | Data Pipeline |
| Bob Smith | 85,000 | API Redesign |
| Bob Smith | 85,000 | Mobile App |
| David Lee | 110,000 | API Redesign |
| David Lee | 110,000 | Data Pipeline |

**7 rows** for 3 employees. Alice appears 3 times (her 3 project assignments). Bob appears twice. David appears twice. Now:

- `COUNT(*)` = 7 (counts every row)
- `SUM(salary)` = 95000×3 + 85000×2 + 110000×2 = 285000 + 170000 + 220000 = **675,000**
- `COUNT(a.project_id)` = 7 (coincidentally correct — there are 7 assignment rows for Engineering)

The salary numbers are inflated because each employee's salary is summed once per assignment row, not once per employee.

**3. Why Marketing escapes:**

Carol has exactly **1** assignment (Marketing Site). Frank has exactly **1** assignment (Marketing Site). Neither employee is duplicated by the JOIN because they each have a single assignment. So `COUNT(*)` = 2 and `SUM(salary)` = 145,000 are accidentally correct. The inflation bug only manifests when employees have **multiple** assignments. Marketing's correct result is a **coincidence of data**, not correct query logic.

**The Key Insight:** JOINs compute row combinations. If employee A has 3 assignments, she appears in 3 rows after the JOIN. Any aggregate (`SUM`, `COUNT`, `AVG`) on employee-level data will be inflated by the fan-out from the assignment JOIN. The fix is to aggregate at the right level first (using CTEs or subqueries), then combine the results.

---

## Solution 1.4: The NOT IN Black Hole

**1. Expected:** Sales (department id 4 has no employees).

**2. Actual:** **0 rows.** Nothing is returned. Not even Sales.

**3. Step-by-step evaluation:**

The subquery returns: `{1, 1, 2, 1, 3, 2, NULL}` (one value per employee row, including Grace's NULL).

For Sales (id = 4), SQL evaluates `4 NOT IN (1, 1, 2, 1, 3, 2, NULL)`. `NOT IN` is equivalent to a chain of `AND`-ed inequalities:

```
4 != 1  → TRUE
4 != 1  → TRUE
4 != 2  → TRUE
4 != 1  → TRUE
4 != 3  → TRUE
4 != 2  → TRUE
4 != NULL → NULL
```

Combined: `TRUE AND TRUE AND TRUE AND TRUE AND TRUE AND TRUE AND NULL` → **`NULL`**

`NULL` is not `TRUE`, so the WHERE condition fails. Sales is excluded.

The same logic applies to **every** department — none can pass the filter because the final `AND NULL` always forces the result to `NULL`. Every row evaluates to `NULL`. The query returns nothing.

**The Fix (two approaches):**

```sql
-- Fix 1: Filter NULLs from the subquery
SELECT name FROM departments
WHERE id NOT IN (
    SELECT department_id FROM employees
    WHERE department_id IS NOT NULL
);

-- Fix 2: Use NOT EXISTS (immune to NULLs)
SELECT d.name FROM departments d
WHERE NOT EXISTS (
    SELECT 1 FROM employees e
    WHERE e.department_id = d.id
);
```

**The Key Insight:** `NOT IN` is unsafe whenever the subquery might return `NULL`. A single `NULL` in the list poisons every comparison and returns zero rows — silently. `NOT EXISTS` avoids this because it checks for the **existence** of matching rows rather than comparing values. As a rule: **prefer `NOT EXISTS` over `NOT IN` unless you can guarantee the subquery has no NULLs.**

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Dashboard That Lies

**Context:** A developer built a "Department Performance Dashboard" query. It runs without errors and returns plausible-looking results. The CTO glances at it and says, "Engineering's payroll is 675K? That can't be right — we only have three engineers."

```sql
-- Department Performance Dashboard
SELECT
    d.name AS department,
    COUNT(*) AS team_size,
    SUM(e.salary) AS total_payroll,
    COUNT(a.project_id) AS total_assignments
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
LEFT JOIN assignments a ON e.id = a.employee_id
GROUP BY d.id, d.name
ORDER BY total_payroll DESC NULLS LAST;
```

**Output the developer sees:**

```
  department  | team_size | total_payroll | total_assignments
──────────────┼───────────┼───────────────┼───────────────────
 Engineering  │         7 │        675000 │                 7
 Design       │         2 │        160000 │                 2
 Marketing    │         2 │        145000 │                 2
 Sales        │         1 │          NULL │                 0
(4 rows)
```

**The correct dashboard should show:**

```
  department  | team_size | total_payroll | total_assignments
──────────────┼───────────┼───────────────┼───────────────────
 Engineering  │         3 │        290000 │                 7
 Marketing    │         2 │        145000 │                 2
 Design       │         1 │         80000 │                 2
 Sales        │         0 │             0 │                 0
(4 rows)
```

---

**Task 2a: Identify the Flaws**

This query has **three distinct bugs** producing **five wrong values**. Identify each bug by comparing the actual output to the correct output. For each wrong value, name the specific SQL concept that causes the error.

Do NOT fix anything yet. Just diagnose.

---

**Task 2b: Explain the Mechanism**

For each bug:

1. **The team_size inflation (Engineering: 7 instead of 3, Design: 2 instead of 1):** Explain exactly how the double LEFT JOIN creates row duplication, and why `COUNT(*)` amplifies it.

2. **The total_payroll inflation (Engineering: 675000 instead of 290000):** Show the arithmetic. How many times is Alice's salary counted? Bob's? David's?

3. **The Sales phantom (team_size: 1 instead of 0, total_payroll: NULL instead of 0):** Explain why LEFT JOIN creates a row for Sales and how `COUNT(*)` treats it differently from `COUNT(e.id)`.

---

**Task 2c: Instrument to Prove Diagnosis**

Before fixing, prove that JOIN multiplication is occurring by running this diagnostic query:

```sql
-- Remove the GROUP BY to see the raw joined rows
-- Add this query and count the rows for Engineering
SELECT d.name, e.name AS employee, e.salary, a.project_id
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
LEFT JOIN assignments a ON e.id = a.employee_id
WHERE d.name = 'Engineering'
ORDER BY e.name, a.project_id;
```

Run it and verify:
- How many rows appear for Alice? For Bob? For David?
- Does the total match the inflated `team_size` of 7?

Then check Sales:
```sql
SELECT d.name, e.name, e.salary, a.project_id
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
LEFT JOIN assignments a ON e.id = a.employee_id
WHERE d.name = 'Sales';
```

What does this row look like? Why does `COUNT(*)` count it?

---

**Task 2d: Fix the Code**

Rewrite the dashboard query so that:
- `team_size` = number of **unique** employees per department
- `total_payroll` = sum of each employee's salary counted **exactly once**
- `total_assignments` = number of assignment rows (this was actually correct)
- Sales shows 0/0/0, not 1/NULL/0
- All four departments appear

You have two approaches. Choose one (or implement both):
- **Approach A:** Use `COUNT(DISTINCT ...)` and a correlated subquery for payroll
- **Approach B:** Use CTEs to aggregate at the right level before joining

---

**Task 2e: Write Verification Tests**

Write SQL queries that produce `PASS` or `FAIL` for each assertion:

```sql
-- Save your fixed query result
DROP TABLE IF EXISTS dashboard;
CREATE TEMP TABLE dashboard AS
<your_fixed_query>;

-- Test 1: Engineering team size is exactly 3
SELECT CASE
    WHEN (SELECT team_size FROM dashboard WHERE department = 'Engineering') = 3
    THEN '✅ PASS: Engineering team_size = 3'
    ELSE '❌ FAIL: Engineering team_size wrong'
END AS test_1;

-- Write tests for:
-- Test 2: Engineering total_payroll = 290000
-- Test 3: Design team_size = 1 (not 2)
-- Test 4: Sales team_size = 0 (not 1)
-- Test 5: Sales total_payroll = 0 (not NULL)
-- Test 6: Total of all team_sizes = 6 (not 12)
```

---

## Exercise 2.2: The Silently Wrong Join

**Context:** A new team member wrote a query to display projects with their team members and department names. The tech lead runs it and says, "This looks fine," then a project manager says, "Wait — where's Eve? And why is Bob listed under Marketing?"

```sql
SELECT
    p.name AS project,
    e.name AS employee,
    d.name AS department,
    a.role
FROM projects p
INNER JOIN assignments a ON p.id = a.project_id
INNER JOIN employees e ON a.employee_id = e.id
INNER JOIN departments d ON e.id = d.id      -- Line 7
ORDER BY p.name, a.role;
```

**Output:**

```
    project     |  employee   | department  |    role
────────────────┼─────────────┼─────────────┼─────────────
 API Redesign   │ Bob Smith   │ Marketing   │ developer
 API Redesign   │ Alice Chen  │ Engineering │ lead
 API Redesign   │ David Lee   │ Sales       │ reviewer
 Data Pipeline  │ Alice Chen  │ Engineering │ developer
 Data Pipeline  │ David Lee   │ Sales       │ lead
 Marketing Site │ Carol Jones │ Design      │ lead
 Mobile App     │ Alice Chen  │ Engineering │ developer
 Mobile App     │ Bob Smith   │ Marketing   │ lead
(8 rows)
```

---

**Task 2a: Identify the Flaw**

1. How many rows should this query return? (Hint: how many rows are in the `assignments` table?)
2. Which employees are **missing** from the output? Why?
3. Which employees show the **wrong department**? Which departments are they actually in?
4. Which single line of SQL contains the bug?

---

**Task 2b: Explain the Mechanism**

Line 7 reads: `INNER JOIN departments d ON e.id = d.id`

1. What does this ON condition actually match? (What does `e.id = d.id` mean in terms of data?)
2. Why does Alice show the **correct** department?
3. Why does Bob show "Marketing" instead of "Engineering"?
4. Why do Eve and Frank disappear entirely?

---

**Task 2c: Instrument to Prove Diagnosis**

Add columns to the query that expose the bug:

```sql
SELECT
    p.name AS project,
    e.name AS employee,
    e.id AS employee_id,          -- Add this
    e.department_id AS actual_dept_id,  -- Add this
    d.id AS joined_dept_id,       -- Add this
    d.name AS department,
    a.role
FROM projects p
INNER JOIN assignments a ON p.id = a.project_id
INNER JOIN employees e ON a.employee_id = e.id
INNER JOIN departments d ON e.id = d.id
ORDER BY p.name, a.role;
```

Run the instrumented query. For each row, compare `actual_dept_id` with `joined_dept_id`. What pattern do you see?

---

**Task 2d: Fix the Code**

Fix Line 7. The fix is a single change to the ON condition.

---

**Task 2e: Write Verification Tests**

```sql
DROP TABLE IF EXISTS project_teams;
CREATE TEMP TABLE project_teams AS
<your_fixed_query>;

-- Test 1: Should return 11 rows (one per assignment)
-- Test 2: Eve Wilson should appear in at least one row
-- Test 3: Bob Smith's department should be 'Engineering' in every row
-- Test 4: No employee should ever show 'Sales' as department
--         (no employee is in the Sales department)
```

Write these as executable SQL that outputs PASS/FAIL.

---

# Level 2 Solutions

## Solution 2.1: The Dashboard That Lies

**Task 2a: Three Bugs, Five Wrong Values**

| Wrong Value | Actual | Correct | Bug |
|---|---|---|---|
| Engineering team_size | 7 | 3 | `COUNT(*)` counts duplicated rows from JOIN multiplication |
| Engineering total_payroll | 675000 | 290000 | `SUM(salary)` adds duplicated salary values from JOIN multiplication |
| Design team_size | 2 | 1 | Eve appears twice (2 assignments) — same JOIN multiplication |
| Sales team_size | 1 | 0 | `COUNT(*)` counts the LEFT JOIN NULL row |
| Sales total_payroll | NULL | 0 | No `COALESCE` to convert NULL to 0 |

**Task 2b: The Mechanisms**

**Bug 1 & 2 — JOIN Multiplication:**

The LEFT JOIN chain `departments → employees → assignments` duplicates employee rows. For Engineering:

```
Alice  (95k)  × 3 assignments = 3 rows of 95,000
Bob    (85k)  × 2 assignments = 2 rows of 85,000
David  (110k) × 2 assignments = 2 rows of 110,000
                                ─────────────────
                         Total: 7 rows, SUM = 675,000
```

`COUNT(*)` sees 7 rows. `SUM(salary)` adds `95000 + 95000 + 95000 + 85000 + 85000 + 110000 + 110000 = 675000`.

**Bug 3 — The Sales Phantom:**

After `departments LEFT JOIN employees`, Sales gets one row with all employee columns set to NULL. After `LEFT JOIN assignments`, that NULL row persists. `COUNT(*)` counts this row as 1. `SUM(NULL)` returns NULL (not 0).

**Task 2c: Diagnostic Output**

Engineering diagnostic:
```
    name     |  employee   | salary | project_id
─────────────┼─────────────┼────────┼────────────
 Engineering │ Alice Chen  │  95000 │          1
 Engineering │ Alice Chen  │  95000 │          2
 Engineering │ Alice Chen  │  95000 │          4
 Engineering │ Bob Smith   │  85000 │          1
 Engineering │ Bob Smith   │  85000 │          2
 Engineering │ David Lee   │ 110000 │          1
 Engineering │ David Lee   │ 110000 │          4
(7 rows)
```

7 rows confirms the inflation. Alice's salary appears 3 times.

Sales diagnostic:
```
  name  | employee | salary | project_id
────────┼──────────┼────────┼────────────
 Sales  │ NULL     │   NULL │       NULL
(1 row)
```

One phantom row. `COUNT(*)` counts it as 1.

**Task 2d: Fixed Query (CTE Approach)**

```sql
WITH dept_employees AS (
    SELECT
        department_id,
        COUNT(*) AS team_size,
        SUM(salary) AS total_payroll
    FROM employees
    WHERE department_id IS NOT NULL
    GROUP BY department_id
),
dept_assignments AS (
    SELECT
        e.department_id,
        COUNT(*) AS total_assignments
    FROM assignments a
    INNER JOIN employees e ON a.employee_id = e.id
    WHERE e.department_id IS NOT NULL
    GROUP BY e.department_id
)
SELECT
    d.name AS department,
    COALESCE(de.team_size, 0) AS team_size,
    COALESCE(de.total_payroll, 0) AS total_payroll,
    COALESCE(da.total_assignments, 0) AS total_assignments
FROM departments d
LEFT JOIN dept_employees de ON d.id = de.department_id
LEFT JOIN dept_assignments da ON d.id = da.department_id
ORDER BY total_payroll DESC;
```

**Why this works:** Each CTE aggregates at the **correct level** before joining. `dept_employees` counts and sums once per employee. `dept_assignments` counts once per assignment. The final query just combines pre-aggregated results — no row duplication is possible.

**Task 2e: Verification Tests**

```sql
DROP TABLE IF EXISTS dashboard;
CREATE TEMP TABLE dashboard AS
<fixed_query_above>;

SELECT CASE
    WHEN (SELECT team_size FROM dashboard WHERE department = 'Engineering') = 3
    THEN '✅ PASS: Engineering team_size = 3'
    ELSE '❌ FAIL: Engineering team_size is ' ||
         (SELECT team_size FROM dashboard WHERE department = 'Engineering')::TEXT
END AS test_1;

SELECT CASE
    WHEN (SELECT total_payroll FROM dashboard WHERE department = 'Engineering') = 290000
    THEN '✅ PASS: Engineering payroll = 290000'
    ELSE '❌ FAIL: Engineering payroll is ' ||
         (SELECT total_payroll FROM dashboard WHERE department = 'Engineering')::TEXT
END AS test_2;

SELECT CASE
    WHEN (SELECT team_size FROM dashboard WHERE department = 'Design') = 1
    THEN '✅ PASS: Design team_size = 1'
    ELSE '❌ FAIL: Design team_size is ' ||
         (SELECT team_size FROM dashboard WHERE department = 'Design')::TEXT
END AS test_3;

SELECT CASE
    WHEN (SELECT team_size FROM dashboard WHERE department = 'Sales') = 0
    THEN '✅ PASS: Sales team_size = 0'
    ELSE '❌ FAIL: Sales team_size is ' ||
         (SELECT team_size FROM dashboard WHERE department = 'Sales')::TEXT
END AS test_4;

SELECT CASE
    WHEN (SELECT total_payroll FROM dashboard WHERE department = 'Sales') = 0
    THEN '✅ PASS: Sales payroll = 0'
    ELSE '❌ FAIL: Sales payroll is ' ||
         COALESCE((SELECT total_payroll FROM dashboard WHERE department = 'Sales')::TEXT, 'NULL')
END AS test_5;

SELECT CASE
    WHEN (SELECT SUM(team_size) FROM dashboard) = 6
    THEN '✅ PASS: Total headcount = 6 (excludes Grace)'
    ELSE '❌ FAIL: Total headcount is ' ||
         (SELECT SUM(team_size) FROM dashboard)::TEXT
END AS test_6;
```

---

## Solution 2.2: The Silently Wrong Join

**Task 2a: Identifying the Flaw**

1. The `assignments` table has **11 rows**, so the query should return 11 rows. It returns only **8**.
2. **Eve Wilson** (id=5) and **Frank Brown** (id=6) are missing — their employee IDs have no matching department ID (there is no department 5 or 6).
3. **Bob Smith** shows "Marketing" (should be Engineering). **Carol Jones** shows "Design" (should be Marketing). **David Lee** shows "Sales" (should be Engineering).
4. **Line 7:** `INNER JOIN departments d ON e.id = d.id`

**Task 2b: The Mechanism**

The ON condition `e.id = d.id` matches **employee ID** to **department ID** — two completely unrelated numbers.

| Employee | e.id | Matched d.id | Shown Department | Actual Department |
|---|---|---|---|---|
| Alice Chen | 1 | 1 | Engineering | Engineering ✅ coincidence |
| Bob Smith | 2 | 2 | Marketing | Engineering ❌ |
| Carol Jones | 3 | 3 | Design | Marketing ❌ |
| David Lee | 4 | 4 | Sales | Engineering ❌ |
| Eve Wilson | 5 | ∅ | **excluded** | Design ❌ |
| Frank Brown | 6 | ∅ | **excluded** | Marketing ❌ |

Alice's result is correct **only by coincidence** — her employee ID (1) happens to equal her department ID (1). This is the most dangerous kind of bug: partial correctness that passes a quick visual check.

Eve (id=5) and Frank (id=6) disappear because no department has id 5 or 6, so the INNER JOIN finds no match and drops them.

**Task 2c: Instrumented Output**

```
    project     |  employee   | employee_id | actual_dept_id | joined_dept_id | department
────────────────┼─────────────┼─────────────┼────────────────┼────────────────┼────────────
 API Redesign   │ Alice Chen  │           1 │              1 │              1 │ Engineering
 API Redesign   │ Bob Smith   │           2 │              1 │              2 │ Marketing
 API Redesign   │ David Lee   │           4 │              1 │              4 │ Sales
 ...
```

The pattern: `employee_id` always equals `joined_dept_id`, but `actual_dept_id` often differs. This proves the JOIN is matching on the wrong column.

**Task 2d: The Fix**

Change Line 7 from:
```sql
INNER JOIN departments d ON e.id = d.id
```
to:
```sql
INNER JOIN departments d ON e.department_id = d.id
```

However, this still drops Grace Kim (NULL department_id). If Grace should appear, use `LEFT JOIN` instead.

**Task 2e: Verification Tests**

```sql
DROP TABLE IF EXISTS project_teams;
CREATE TEMP TABLE project_teams AS
SELECT
    p.name AS project,
    e.name AS employee,
    d.name AS department,
    a.role
FROM projects p
INNER JOIN assignments a ON p.id = a.project_id
INNER JOIN employees e ON a.employee_id = e.id
INNER JOIN departments d ON e.department_id = d.id  -- FIXED
ORDER BY p.name, a.role;

-- Test 1: 11 rows (one per assignment)
SELECT CASE
    WHEN (SELECT COUNT(*) FROM project_teams) = 11
    THEN '✅ PASS: 11 rows returned'
    ELSE '❌ FAIL: Expected 11, got ' || (SELECT COUNT(*) FROM project_teams)::TEXT
END AS test_1;

-- Test 2: Eve Wilson present
SELECT CASE
    WHEN EXISTS (SELECT 1 FROM project_teams WHERE employee = 'Eve Wilson')
    THEN '✅ PASS: Eve Wilson is present'
    ELSE '❌ FAIL: Eve Wilson is missing'
END AS test_2;

-- Test 3: Bob Smith always Engineering
SELECT CASE
    WHEN NOT EXISTS (
        SELECT 1 FROM project_teams
        WHERE employee = 'Bob Smith' AND department != 'Engineering'
    )
    THEN '✅ PASS: Bob is always Engineering'
    ELSE '❌ FAIL: Bob shows wrong department'
END AS test_3;

-- Test 4: No employee shows Sales
SELECT CASE
    WHEN NOT EXISTS (SELECT 1 FROM project_teams WHERE department = 'Sales')
    THEN '✅ PASS: No employee in Sales'
    ELSE '❌ FAIL: Someone incorrectly shows Sales'
END AS test_4;
```

---

# LEVEL 2.5: EXTEND EXISTING CODE (The Integration)

## The Employee Directory Report

**Given:** The following working query that produces a correct employee directory:

```sql
-- Base Query: Employee Directory (WORKING — DO NOT BREAK)
WITH employee_info AS (
    SELECT
        e.id,
        e.name,
        e.salary,
        e.hire_date,
        COALESCE(d.name, 'Unassigned') AS department,
        COUNT(a.project_id) AS project_count
    FROM employees e
    LEFT JOIN departments d ON e.department_id = d.id
    LEFT JOIN assignments a ON e.id = a.employee_id
    GROUP BY e.id, e.name, e.salary, e.hire_date, d.name
)
SELECT * FROM employee_info
ORDER BY department, name;
```

**Current output (all values correct):**

```
     name     | salary | hire_date  | department  | project_count
──────────────┼────────┼────────────┼─────────────┼───────────────
 Eve Wilson   │  80000 │ 2023-01-20 │ Design      │             2
 Alice Chen   │  95000 │ 2022-01-15 │ Engineering │             3
 Bob Smith    │  85000 │ 2022-06-01 │ Engineering │             2
 David Lee    │ 110000 │ 2021-09-01 │ Engineering │             2
 Carol Jones  │  75000 │ 2023-03-10 │ Marketing   │             1
 Frank Brown  │  70000 │ 2023-07-15 │ Marketing   │             1
 Grace Kim    │  90000 │ 2024-01-10 │ Unassigned  │             0
(7 rows)
```

**Base Test Suite (must pass before AND after every extension):**

```sql
DROP TABLE IF EXISTS directory;
CREATE TEMP TABLE directory AS <your_extended_query>;

-- Base Test 1: Exactly 7 rows
SELECT CASE WHEN (SELECT COUNT(*) FROM directory) = 7
    THEN '✅ BASE-1 PASS' ELSE '❌ BASE-1 FAIL: row count changed' END;

-- Base Test 2: Alice has 3 projects
SELECT CASE WHEN (SELECT project_count FROM directory WHERE name = 'Alice Chen') = 3
    THEN '✅ BASE-2 PASS' ELSE '❌ BASE-2 FAIL: Alice project_count changed' END;

-- Base Test 3: Grace has 0 projects and is Unassigned
SELECT CASE WHEN (SELECT project_count FROM directory WHERE name = 'Grace Kim') = 0
    AND (SELECT department FROM directory WHERE name = 'Grace Kim') = 'Unassigned'
    THEN '✅ BASE-3 PASS' ELSE '❌ BASE-3 FAIL: Grace data changed' END;

-- Base Test 4: All salaries unchanged
SELECT CASE WHEN (SELECT SUM(salary) FROM directory) = 605000
    THEN '✅ BASE-4 PASS' ELSE '❌ BASE-4 FAIL: salary sum changed' END;
```

---

### Extension 1: Salary Comparison (Easy)

Add a column called `salary_vs_avg` that shows `'Above Average'` if the employee's salary is above or equal to the company-wide average, or `'Below Average'` otherwise.

**Requirements:**
- The company average is calculated across ALL 7 employees
- No existing column values may change
- All base tests must still pass

**Expected new column values:**

| name | salary | salary_vs_avg |
|---|---|---|
| David Lee | 110000 | Above Average |
| Alice Chen | 95000 | Above Average |
| Grace Kim | 90000 | Above Average |
| Bob Smith | 85000 | Below Average |
| Eve Wilson | 80000 | Below Average |
| Carol Jones | 75000 | Below Average |
| Frank Brown | 70000 | Below Average |

**Extension 1 Test:**
```sql
-- Ext1 Test: David is above, Frank is below
SELECT CASE
    WHEN (SELECT salary_vs_avg FROM directory WHERE name = 'David Lee') = 'Above Average'
    AND  (SELECT salary_vs_avg FROM directory WHERE name = 'Frank Brown') = 'Below Average'
    THEN '✅ EXT1 PASS' ELSE '❌ EXT1 FAIL' END;
```

---

### Extension 2: Lead Count (Medium)

Add a column called `lead_count` that shows how many projects each employee **leads** (where `role = 'lead'` in the assignments table).

**Requirements:**
- Employees who lead no projects show 0
- Must NOT inflate `project_count` or any other existing column
- All base tests must still pass

**Expected new column values:**

| name | project_count | lead_count |
|---|---|---|
| Alice Chen | 3 | 1 |
| Bob Smith | 2 | 1 |
| Carol Jones | 1 | 1 |
| David Lee | 2 | 1 |
| Eve Wilson | 2 | 0 |
| Frank Brown | 1 | 0 |
| Grace Kim | 0 | 0 |

**Extension 2 Test:**
```sql
-- Ext2 Test: Alice leads 1 project, Eve leads 0
SELECT CASE
    WHEN (SELECT lead_count FROM directory WHERE name = 'Alice Chen') = 1
    AND  (SELECT lead_count FROM directory WHERE name = 'Eve Wilson') = 0
    AND  (SELECT project_count FROM directory WHERE name = 'Alice Chen') = 3  -- unchanged!
    THEN '✅ EXT2 PASS' ELSE '❌ EXT2 FAIL' END;
```

**Hint:** You can solve this within the existing GROUP BY using conditional aggregation, or with a correlated subquery. One approach is cleaner than the other — can you figure out which?

---

### Extension 3: Department Salary Rank (Hard)

Add a column called `dept_salary_rank` showing the employee's salary rank **within their department** (1 = highest paid in the department).

**Requirements:**
- Rank is per-department, not company-wide
- Grace Kim (Unassigned, no department) should show `NULL` for rank
- Ties: if two employees have the same salary, they get the same rank
- All base tests must still pass
- You may NOT use window functions (`RANK()`, `ROW_NUMBER()`, etc.) — these are not taught until later. Use a correlated subquery.

**Expected new column values:**

| name | department | salary | dept_salary_rank |
|---|---|---|---|
| David Lee | Engineering | 110000 | 1 |
| Alice Chen | Engineering | 95000 | 2 |
| Bob Smith | Engineering | 85000 | 3 |
| Carol Jones | Marketing | 75000 | 1 |
| Frank Brown | Marketing | 70000 | 2 |
| Eve Wilson | Design | 80000 | 1 |
| Grace Kim | Unassigned | 90000 | NULL |

**Extension 3 Test:**
```sql
-- Ext3 Test: Ranks correct
SELECT CASE
    WHEN (SELECT dept_salary_rank FROM directory WHERE name = 'David Lee') = 1
    AND  (SELECT dept_salary_rank FROM directory WHERE name = 'Bob Smith') = 3
    AND  (SELECT dept_salary_rank FROM directory WHERE name = 'Grace Kim') IS NULL
    THEN '✅ EXT3 PASS' ELSE '❌ EXT3 FAIL' END;
```

**Hint:** A correlated subquery that counts how many employees in the same department earn a strictly higher salary, plus 1, gives the rank. But be careful — what does `e2.department_id = e.department_id` evaluate to when `e.department_id` is NULL?

---

# Level 2.5 Solutions

## Extension 1 Solution: Salary Comparison

```sql
WITH employee_info AS (
    SELECT
        e.id,
        e.name,
        e.salary,
        e.hire_date,
        COALESCE(d.name, 'Unassigned') AS department,
        COUNT(a.project_id) AS project_count
    FROM employees e
    LEFT JOIN departments d ON e.department_id = d.id
    LEFT JOIN assignments a ON e.id = a.employee_id
    GROUP BY e.id, e.name, e.salary, e.hire_date, d.name
)
SELECT
    *,
    CASE
        WHEN salary >= (SELECT AVG(salary) FROM employees)
        THEN 'Above Average'
        ELSE 'Below Average'
    END AS salary_vs_avg
FROM employee_info
ORDER BY department, name;
```

**Why this works:** The scalar subquery `(SELECT AVG(salary) FROM employees)` computes the company average (≈86,429) once. The `CASE` expression in the outer SELECT compares each row's salary against this value. No existing columns are affected because we simply added a new computed column to the final SELECT.

---

## Extension 2 Solution: Lead Count

**Clean approach — conditional aggregation within existing GROUP BY:**

```sql
WITH employee_info AS (
    SELECT
        e.id,
        e.name,
        e.salary,
        e.hire_date,
        COALESCE(d.name, 'Unassigned') AS department,
        COUNT(a.project_id) AS project_count,
        COUNT(CASE WHEN a.role = 'lead' THEN 1 END) AS lead_count
    FROM employees e
    LEFT JOIN departments d ON e.department_id = d.id
    LEFT JOIN assignments a ON e.id = a.employee_id
    GROUP BY e.id, e.name, e.salary, e.hire_date, d.name
)
SELECT * FROM employee_info
ORDER BY department, name;
```

**Why this works:** `COUNT(CASE WHEN a.role = 'lead' THEN 1 END)` only counts rows where the CASE expression returns a non-NULL value (i.e., where the role is 'lead'). Rows where the role is 'developer', 'designer', etc. produce NULL from the CASE, and `COUNT` skips NULLs. This adds the new column without adding any new JOINs, so `project_count` and all other columns remain unchanged.

**Alternative (correlated subquery) — works but is less elegant:**

```sql
-- In the final SELECT, replace the CTE approach with:
(SELECT COUNT(*) FROM assignments a2
 WHERE a2.employee_id = e.id AND a2.role = 'lead') AS lead_count
```

---

## Extension 3 Solution: Department Salary Rank

```sql
WITH employee_info AS (
    SELECT
        e.id,
        e.name,
        e.salary,
        e.hire_date,
        e.department_id,
        COALESCE(d.name, 'Unassigned') AS department,
        COUNT(a.project_id) AS project_count,
        COUNT(CASE WHEN a.role = 'lead' THEN 1 END) AS lead_count
    FROM employees e
    LEFT JOIN departments d ON e.department_id = d.id
    LEFT JOIN assignments a ON e.id = a.employee_id
    GROUP BY e.id, e.name, e.salary, e.hire_date, e.department_id, d.name
)
SELECT
    name,
    salary,
    hire_date,
    department,
    project_count,
    lead_count,
    CASE
        WHEN department_id IS NULL THEN NULL
        ELSE (
            SELECT COUNT(*) + 1
            FROM employees e2
            WHERE e2.department_id = employee_info.department_id
              AND e2.salary > employee_info.salary
        )
    END AS dept_salary_rank
FROM employee_info
ORDER BY department, name;
```

**Why the CASE WHEN is necessary:** Without it, Grace's rank calculation would evaluate `e2.department_id = NULL`, which produces `NULL` for every comparison. The correlated subquery would return 0 rows, giving Grace a rank of 1. But Grace isn't *ranked first* in some department — she's *unranked* because she has no department. The CASE explicitly returns NULL for unassigned employees, which is semantically correct.

**How the rank subquery works:** For David Lee (salary 110000, dept 1): "How many employees in department 1 earn MORE than 110000?" Answer: 0. Rank = 0 + 1 = **1**. For Alice (95000, dept 1): "How many earn more than 95000 in dept 1?" Answer: 1 (David). Rank = 1 + 1 = **2**. This naturally handles ties — if two people earn the same amount, the same number of people earn more than both, so they get the same rank.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS (The Application)

## The Organization Workload Report

**Scenario:** Your CTO needs a single, authoritative "Organization Workload Report" query for the quarterly board meeting. The numbers must be **exactly correct** — the board will cross-reference them against HR's payroll system and the project management tool. Any discrepancy will undermine trust in your data infrastructure.

**The naive approach — a single query joining all four tables — will produce wrong numbers** (as you saw in Level 2). You must design a query architecture that avoids JOIN multiplication while still combining data from all tables.

---

### 3a: Implement Core (Basic Structure)

Write a query that produces this **exact** output:

```
  department  | emp_count | total_payroll
──────────────┼───────────┼───────────────
 Engineering  │         3 │        290000
 Marketing    │         2 │        145000
 Design       │         1 │         80000
 Sales        │         0 │             0
(4 rows)
```

**Constraints:**
- All 4 departments must appear (including Sales)
- `emp_count` for Sales must be `0`, not `1` or `NULL`
- `total_payroll` for Sales must be `0`, not `NULL`
- Engineering total_payroll must be `290000`, not `675000`
- Sorted by total_payroll descending

**3a Verification Tests:**

```sql
DROP TABLE IF EXISTS report;
CREATE TEMP TABLE report AS <your_query>;

SELECT CASE WHEN (SELECT COUNT(*) FROM report) = 4
    THEN '✅ 3a-1 PASS: 4 departments'
    ELSE '❌ 3a-1 FAIL' END;

SELECT CASE WHEN (SELECT emp_count FROM report WHERE department = 'Sales') = 0
    THEN '✅ 3a-2 PASS: Sales emp_count = 0'
    ELSE '❌ 3a-2 FAIL' END;

SELECT CASE WHEN (SELECT total_payroll FROM report WHERE department = 'Sales') = 0
    THEN '✅ 3a-3 PASS: Sales payroll = 0'
    ELSE '❌ 3a-3 FAIL' END;

SELECT CASE WHEN (SELECT total_payroll FROM report WHERE department = 'Engineering') = 290000
    THEN '✅ 3a-4 PASS: Engineering payroll = 290000'
    ELSE '❌ 3a-4 FAIL: got ' ||
         (SELECT total_payroll FROM report WHERE department = 'Engineering')::TEXT END;
```

---

### 3b: Add Salary Analytics

Extend your 3a query to include `avg_salary`, `min_salary`, and `max_salary`:

```
  department  | emp_count | total_payroll | avg_salary | min_salary | max_salary
──────────────┼───────────┼───────────────┼────────────┼────────────┼────────────
 Engineering  │         3 │        290000 │      96667 │      85000 │     110000
 Marketing    │         2 │        145000 │      72500 │      70000 │      75000
 Design       │         1 │         80000 │      80000 │      80000 │      80000
 Sales        │         0 │             0 │          0 │          0 │          0
(4 rows)
```

**Constraints:**
- All 3a tests must still pass
- `avg_salary` is rounded to integer (`::INTEGER`)
- Sales shows `0` for all salary columns, not `NULL`
- Engineering `avg_salary` must be `96667` (= ROUND(290000/3)), not an inflated number

**3b Additional Tests:**

```sql
SELECT CASE WHEN (SELECT avg_salary FROM report WHERE department = 'Engineering') = 96667
    THEN '✅ 3b-1 PASS: Engineering avg correct'
    ELSE '❌ 3b-1 FAIL' END;

SELECT CASE WHEN (SELECT max_salary FROM report WHERE department = 'Engineering') = 110000
    THEN '✅ 3b-2 PASS: Engineering max correct'
    ELSE '❌ 3b-2 FAIL' END;

SELECT CASE WHEN (SELECT min_salary FROM report WHERE department = 'Sales') = 0
    THEN '✅ 3b-3 PASS: Sales min = 0'
    ELSE '❌ 3b-3 FAIL' END;
```

---

### 3c: Add Project Statistics Without Breaking Salary Numbers

Add `project_count` (number of **distinct** projects the department is involved in) and `unassigned_projects` (number of projects that have **zero** team members assigned, company-wide — same value for every row, as context for the CTO):

```
  department  | emp_count | total_payroll | avg_salary | min_salary | max_salary | project_count | unassigned_projects
──────────────┼───────────┼───────────────┼────────────┼────────────┼────────────┼───────────────┼─────────────────────
 Engineering  │         3 │        290000 │      96667 │      85000 │     110000 │             3 │                   1
 Marketing    │         2 │        145000 │      72500 │      70000 │      75000 │             1 │                   1
 Design       │         1 │         80000 │      80000 │      80000 │      80000 │             2 │                   1
 Sales        │         0 │             0 │          0 │          0 │          0 │             0 │                   1
(4 rows)
```

**Constraints:**
- All 3a and 3b tests must still pass — if adding project data breaks the salary numbers, your approach is wrong
- `project_count` counts **distinct** projects (Engineering employees work on 3 distinct projects: API Redesign, Mobile App, Data Pipeline)
- `unassigned_projects` = 1 (Brand Refresh has no assignments)
- Must be a single query (CTEs are encouraged)

**3c Additional Tests:**

```sql
-- Projects correct
SELECT CASE WHEN (SELECT project_count FROM report WHERE department = 'Engineering') = 3
    THEN '✅ 3c-1 PASS: Engineering has 3 projects'
    ELSE '❌ 3c-1 FAIL' END;

SELECT CASE WHEN (SELECT project_count FROM report WHERE department = 'Design') = 2
    THEN '✅ 3c-2 PASS: Design has 2 projects'
    ELSE '❌ 3c-2 FAIL' END;

SELECT CASE WHEN (SELECT project_count FROM report WHERE department = 'Sales') = 0
    THEN '✅ 3c-3 PASS: Sales has 0 projects'
    ELSE '❌ 3c-3 FAIL' END;

-- Payroll STILL correct after adding projects
SELECT CASE WHEN (SELECT total_payroll FROM report WHERE department = 'Engineering') = 290000
    THEN '✅ 3c-4 PASS: Payroll survived extension'
    ELSE '❌ 3c-4 FAIL: Adding projects broke payroll!' END;

-- Unassigned projects
SELECT CASE WHEN (SELECT DISTINCT unassigned_projects FROM report) = 1
    THEN '✅ 3c-5 PASS: 1 unassigned project'
    ELSE '❌ 3c-5 FAIL' END;
```

---

# Level 3 Solutions

## Solution 3a: Core Report

```sql
SELECT
    d.name AS department,
    COUNT(e.id) AS emp_count,
    COALESCE(SUM(e.salary), 0) AS total_payroll
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
GROUP BY d.id, d.name
ORDER BY total_payroll DESC;
```

**Why this works:** This query joins **only** departments and employees — no assignment table, so no row multiplication. `COUNT(e.id)` counts non-NULL employee IDs (returns 0 for Sales, not 1 like `COUNT(*)` would). `COALESCE(SUM(...), 0)` converts NULL (Sales, no employees to sum) to 0.

This is the cleanest possible foundation. The key insight is: **don't join the assignments table at this stage**. Salary data lives in `employees` only, so the JOIN should stop there.

---

## Solution 3b: Salary Analytics

```sql
SELECT
    d.name AS department,
    COUNT(e.id) AS emp_count,
    COALESCE(SUM(e.salary), 0) AS total_payroll,
    COALESCE(AVG(e.salary)::INTEGER, 0) AS avg_salary,
    COALESCE(MIN(e.salary), 0) AS min_salary,
    COALESCE(MAX(e.salary), 0) AS max_salary
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
GROUP BY d.id, d.name
ORDER BY total_payroll DESC;
```

**Why this works:** All aggregate functions (`AVG`, `MIN`, `MAX`) operate on the same un-duplicated employee rows. Since we haven't joined assignments, there's no inflation. `COALESCE(..., 0)` handles the Sales department uniformly.

---

## Solution 3c: Adding Project Statistics

```sql
WITH dept_stats AS (
    SELECT
        d.id AS dept_id,
        d.name AS department,
        COUNT(e.id) AS emp_count,
        COALESCE(SUM(e.salary), 0) AS total_payroll,
        COALESCE(AVG(e.salary)::INTEGER, 0) AS avg_salary,
        COALESCE(MIN(e.salary), 0) AS min_salary,
        COALESCE(MAX(e.salary), 0) AS max_salary
    FROM departments d
    LEFT JOIN employees e ON d.id = e.department_id
    GROUP BY d.id, d.name
),
dept_projects AS (
    SELECT
        e.department_id,
        COUNT(DISTINCT a.project_id) AS project_count
    FROM employees e
    INNER JOIN assignments a ON e.id = a.employee_id
    WHERE e.department_id IS NOT NULL
    GROUP BY e.department_id
),
orphan_projects AS (
    SELECT COUNT(*) AS unassigned_projects
    FROM projects p
    WHERE NOT EXISTS (
        SELECT 1 FROM assignments a WHERE a.project_id = p.id
    )
)
SELECT
    ds.department,
    ds.emp_count,
    ds.total_payroll,
    ds.avg_salary,
    ds.min_salary,
    ds.max_salary,
    COALESCE(dp.project_count, 0) AS project_count,
    op.unassigned_projects
FROM dept_stats ds
LEFT JOIN dept_projects dp ON ds.dept_id = dp.department_id
CROSS JOIN orphan_projects op
ORDER BY ds.total_payroll DESC;
```

**Why this works:** Three CTEs, each aggregating at the correct level:

`dept_stats` aggregates employees per department — no assignment table, no inflation. This is the 3b query wrapped in a CTE. `dept_projects` aggregates assignments per department, counting `DISTINCT` project IDs to avoid counting the same project multiple times. This is computed separately from salary data, so it cannot inflate payroll numbers. `orphan_projects` counts projects with no assignments at all (Brand Refresh). This is a single scalar value, applied to every row via `CROSS JOIN`.

The final SELECT combines pre-aggregated CTEs with LEFT JOINs. Since each CTE produces one row per department (already collapsed), the JOINs cannot multiply rows.

**The architectural lesson:** When a report requires data from tables at different levels of granularity (employee-level salaries, assignment-level project counts, project-level orphan counts), **aggregate each level separately in CTEs, then combine the pre-aggregated results**. Never aggregate after joining tables at different granularities.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE (The Distinction)

## When SQL Normalization Makes Things Worse

The entire lecture argues that SQL and normalized relational tables are superior to Python lists and dictionaries. Now argue the opposite.

**Task:**

1. **Describe a concrete, realistic scenario** where storing data in a single denormalized table (or a JSON/JSONB column, or even an in-memory Python dict) is **better** than normalizing it into separate tables with foreign keys and JOINs.

2. **Write two implementations** of the same query for your scenario: one using normalized tables with JOINs, and one using a denormalized approach. Show where the normalized version is unnecessarily complex or slower.

3. **State the tradeoff explicitly.** What do you **gain** from denormalization? What do you **lose**?

**Constraint:** Your scenario must be realistic (something a backend developer might actually encounter), not a contrived edge case. "A table with one row" doesn't count.

---

# Level 4 Solution

**Scenario: User Preferences / Settings**

A user has a set of preferences: theme, language, timezone, notification settings, dashboard layout. These preferences are unstructured, vary per user, rarely queried across users, and change frequently.

**Normalized approach (3 tables):**

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE preference_keys (
    id SERIAL PRIMARY KEY,
    key_name TEXT NOT NULL UNIQUE
);

CREATE TABLE user_preferences (
    user_id INTEGER REFERENCES users(id),
    pref_id INTEGER REFERENCES preference_keys(id),
    value TEXT NOT NULL,
    PRIMARY KEY (user_id, pref_id)
);

-- "Get all preferences for user 42"
SELECT pk.key_name, up.value
FROM user_preferences up
INNER JOIN preference_keys pk ON up.pref_id = pk.id
WHERE up.user_id = 42;

-- "Update user 42's theme"
UPDATE user_preferences
SET value = 'dark'
WHERE user_id = 42
AND pref_id = (SELECT id FROM preference_keys WHERE key_name = 'theme');
```

**Denormalized approach (JSONB column):**

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    preferences JSONB NOT NULL DEFAULT '{}'
);

-- "Get all preferences for user 42"
SELECT preferences FROM users WHERE id = 42;

-- "Update user 42's theme"
UPDATE users
SET preferences = jsonb_set(preferences, '{theme}', '"dark"')
WHERE id = 42;
```

**What you gain from denormalization:** A single row read returns all preferences atomically. No JOINs. Adding a new preference type requires no schema migration. The read path is a simple primary key lookup. The write path is a single UPDATE. This maps naturally to how the application uses the data: "load user, get their settings blob, render the page."

**What you lose:** You cannot efficiently query "which users have theme = 'dark'?" across all users without scanning every row (unless you add a GIN index on the JSONB column). There's no referential integrity — a typo in a key name is stored silently. You cannot enforce that "theme" must be one of ['light', 'dark', 'system'] at the database level.

**The rule of thumb:** Normalize data that has **relationships, is queried across entities, or needs integrity constraints**. Denormalize data that is **entity-scoped, schema-flexible, and read as a unit**.

---

# LEVEL 5: THE TEACH BACK (Synthesis)

## "LEFT JOIN Is Always Safer"

**Scenario:** A junior developer on your team submits a PR where every single JOIN is a LEFT JOIN. During code review, you ask why. They respond:

> "I always use LEFT JOIN instead of INNER JOIN because LEFT JOIN is safer. INNER JOIN silently drops rows — you could lose data and never notice. LEFT JOIN keeps everything, so you never accidentally hide information. There's no reason to ever use INNER JOIN."

**Task:**

1. **Acknowledge the valid concern** in their reasoning. What real danger of INNER JOIN are they correctly worried about?

2. **Explain why their blanket rule is wrong.** Give a specific example using the lecture dataset where using LEFT JOIN instead of INNER JOIN produces **incorrect or misleading results**. Show the actual output.

3. **Give them a decision framework.** In 3-4 sentences, explain when to choose INNER JOIN vs LEFT JOIN, phrased as a question the developer should ask themselves before writing each JOIN.

---

# Level 5 Solution

**1. The valid concern:**

They are right that INNER JOIN can silently drop rows. If a foreign key is nullable (like `employees.department_id`) or if data integrity is imperfect (orphaned records), INNER JOIN will exclude those rows without warning. The result looks correct but is incomplete. This is a real and common source of bugs, especially in messy production databases. Their instinct to be cautious about data loss is good.

**2. Why the blanket rule is wrong:**

Using LEFT JOIN blindly introduces a different class of bugs — phantom NULLs that corrupt aggregations.

Consider the CTO's dashboard from Exercise 2.1. If the developer writes:

```sql
SELECT
    d.name AS department,
    COUNT(*) AS team_size
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
GROUP BY d.id, d.name;
```

Sales shows `team_size = 1` because `COUNT(*)` counts the NULL row produced by the LEFT JOIN. The developer's "safe" approach just told the CTO that Sales has an employee when it doesn't. If they had used INNER JOIN, Sales wouldn't appear at all — which, depending on the business question, might be the **correct** behavior (the report is about staffed departments).

Worse: if the developer chains a LEFT JOIN to the assignments table and aggregates salary, Engineering's payroll inflates to 675,000 and nobody notices because "LEFT JOIN keeps everything" — including duplicate rows the developer didn't realize were there.

The blanket LEFT JOIN approach trades one silent error (missing rows) for another (phantom rows and inflated aggregations). Neither is "safe." Both require understanding the data model.

**3. The decision framework:**

Before writing each JOIN, ask: "Should rows from the left table appear in the result **even if they have no match** on the right?" If the answer is yes — for example, "show all departments, including empty ones" — use LEFT JOIN and handle the NULLs explicitly in your aggregations (use `COUNT(column)`, not `COUNT(*)`; use `COALESCE` for sums). If the answer is no — for example, "show employees with their department names" where an employee without a department is a data error — use INNER JOIN, because including a NULL-filled row would be misleading. The JOIN type should reflect the **business logic** of the question, not a blanket safety rule.