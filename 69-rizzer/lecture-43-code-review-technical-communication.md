# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PAIN FIRST, PRINCIPLES SECOND                                  │
│  ─────────────────────────────                                  │
│  Students will READ bad code, bad PRs, bad docs — and FEEL      │
│  the frustration. Then we teach them how to never inflict that  │
│  pain on others.                                                │
│                                                                 │
│  THEIR CODE, THEIR STACK                                        │
│  ────────────────────────                                       │
│  Every example uses FastAPI, SQLAlchemy, Pydantic, Celery —     │
│  the tools they've used for 13 weeks. Nothing hypothetical.     │
│                                                                 │
│  THIS IS ENGINEERING, NOT "SOFT SKILLS"                         │
│  ──────────────────────────────────────                         │
│  Code that cannot be understood cannot be maintained.            │
│  A PR that cannot be reviewed cannot be trusted.                │
│  A system that cannot be documented cannot be operated.          │
│  Communication IS the engineering.                               │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Git & Conventional Commits (Wk1) → PR authoring, git history  │
│  Testing (Wk2) → test coverage as review criterion              │
│  Pydantic (Wk3) → validation review, schema review             │
│  Repository Pattern (Wk6) → separation of concerns in review   │
│  Multi-tenancy (Wk13 L2) → tenant isolation as review concern  │
│  ADRs (Wk13 L1) → architecture documentation                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│              CODE REVIEW & TECHNICAL COMMUNICATION              │
│                      (3–4 Hour Lecture)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (30 min)                                   │
│  ├─ 1.1 The 60-Second Review (Demonstration)                    │
│  ├─ 1.2 Code Is Read More Than Written                          │
│  ├─ 1.3 The Building Analogy                                    │
│  └─ 1.4 Your Three Audiences                                    │
│                                                                 │
│  PART 2: CODE REVIEW — THE REVIEWER'S CRAFT (55 min)            │
│  ├─ 2.1 The Review Pyramid                                      │
│  ├─ 2.2 What to Catch: Architecture, Logic, Security            │
│  ├─ 2.3 How to Say It: Feedback That Helps                      │
│  └─ 2.4 A Full Review Walkthrough                               │
│                                                                 │
│  PART 3: CODE REVIEW — THE AUTHOR'S CRAFT (45 min)              │
│  ├─ 3.1 Writing Reviewable PRs                                  │
│  ├─ 3.2 The PR Description                                      │
│  ├─ 3.3 Commit History as Narrative (Week 1 Connection)         │
│  └─ 3.4 Receiving Feedback Without Ego                          │
│                                                                 │
│  PART 4: TECHNICAL DOCUMENTATION (45 min)                       │
│  ├─ 4.1 The Documentation Stack                                 │
│  ├─ 4.2 README: The Front Door                                  │
│  ├─ 4.3 Runbooks: The 3 AM Guide                                │
│  └─ 4.4 Inline Documentation: When and How                      │
│                                                                 │
│  PART 5: ESTIMATION & NAVIGATING CODEBASES (30 min)             │
│  ├─ 5.1 Breaking Down Features Into Tasks                       │
│  ├─ 5.2 Why Estimates Fail (And How to Improve)                 │
│  ├─ 5.3 Reading Code You Didn't Write                           │
│  └─ 5.4 Your First Contribution to an Existing Codebase         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The 60-Second Review

**You're a reviewer. You open a pull request. The title is "updates." No description. You see this endpoint:**

```python
@router.post("/tasks/{task_id}/assign")
async def assign_task(
    task_id: int,
    data: dict,
    db: AsyncSession = Depends(get_db),
):
    task = await db.execute(select(Task).where(Task.id == task_id))
    task = task.scalar_one_or_none()
    if not task:
        raise HTTPException(status_code=400, detail="not found")

    user = await db.execute(select(User).where(User.id == data["user_id"]))
    user = user.scalar_one_or_none()

    task.assigned_to = user.id
    task.status = "assigned"
    task.updated_at = datetime.now()
    await db.commit()

    return {"message": "ok"}
```

**Now ask yourself: would you approve this?**

Take 60 seconds. Count the issues. Write down everything wrong.

> "Done? Let's see what you found. And what you missed."

**The issues — in order of severity:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE ISSUE LIST                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  🔴 CRITICAL (Security / Correctness)                           │
│  ├─ No authentication — any anonymous request can reassign      │
│  │   tasks. Where is get_current_user?                          │
│  ├─ No tenant isolation — user from Org A can reassign a        │
│  │   task belonging to Org B. (Week 13 Lecture 2: multi-tenant) │
│  ├─ data: dict — no Pydantic model. No validation at all.       │
│  │   What if data has no "user_id"? KeyError in production.     │
│  └─ Line 12: if user is None, user.id raises AttributeError.   │
│     There is no None check on the user query result.            │
│                                                                 │
│  🟡 BUGS (Logic Errors)                                         │
│  ├─ status_code=400 for "not found" — should be 404.            │
│  ├─ datetime.now() — naive local time. The rest of our models   │
│  │   use datetime.now(UTC). This will create timezone bugs.     │
│  └─ No audit log entry. Capstone requirement: log mutations.    │
│                                                                 │
│  🟠 DESIGN (Maintainability)                                    │
│  ├─ Route handler queries DB directly — bypasses service layer. │
│  │   No separation of concerns. (Week 6: repository pattern)    │
│  ├─ Returns bare dict {"message": "ok"} — no response model.   │
│  │   Swagger docs will show nothing useful.                     │
│  ├─ Magic string "assigned" — should be a TaskStatus enum.      │
│  └─ No type hints on return. mypy will not catch anything.      │
│                                                                 │
│  🔵 MINOR (Style)                                               │
│  └─ Inconsistent with project conventions (but least important) │
│                                                                 │
│  TOTAL: 11 issues in 15 lines of code.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The point is not that this code is terrible. The point is:**

> "You've been writing code for 13 weeks. You KNOW all of these things. You've used Pydantic since Week 3, dependency injection since Week 3, tenant isolation since Lecture 2 of this week. But when you read someone else's code quickly, it's easy to miss issues unless you have a SYSTEMATIC approach. That's what code review is. And it's easy to WRITE this code yourself on a tired afternoon. That's why we review."

---

## 1.2 Code Is Read More Than Written

**The fundamental asymmetry of software:**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE READING-WRITING ASYMMETRY                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You write a function ONCE.                                     │
│                                                                 │
│  That function is read by:                                      │
│  ├─ Your reviewer (this week)                                   │
│  ├─ Your teammate who calls it (next week)                      │
│  ├─ You, debugging a production issue (next month)              │
│  ├─ A new team member trying to understand the system (next Q)  │
│  ├─ The on-call engineer at 3 AM (someday)                      │
│  └─ You again, completely confused by your own code (6 months)  │
│                                                                 │
│  WRITE ONCE.  READ TEN TIMES.  MAYBE FIFTY.                    │
│                                                                 │
│  Every minute you save by writing code hastily                  │
│  costs ten minutes of someone else's confusion later.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**This has a concrete implication:**

> "When you write code, you are not writing it for the computer. The computer doesn't care. You are writing it for the NEXT HUMAN who will read it. That human might be you. Optimize for reading, not for writing."

This changes how you think about naming, structure, comments, PR descriptions — everything in this lecture.

---

## 1.3 The Building Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE BUILDING ANALOGY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your codebase is a BUILDING.                                   │
│                                                                 │
│  ┌──────────────────────────┬──────────────────────────────┐    │
│  │  Building                │  Software                     │    │
│  ├──────────────────────────┼──────────────────────────────┤    │
│  │  Building inspection     │  Code review                  │    │
│  │  Permit application      │  Pull request description     │    │
│  │  Architectural blueprint │  Architecture Decision Record │    │
│  │  Welcome guide / map     │  README                       │    │
│  │  Emergency procedures    │  Runbook                      │    │
│  │  Construction estimate   │  Task estimation               │    │
│  │  Studying an existing    │  Reading an existing           │    │
│  │    building before       │    codebase before             │    │
│  │    renovating            │    contributing                │    │
│  └──────────────────────────┴──────────────────────────────┘    │
│                                                                 │
│  No one builds a skyscraper without blueprints.                 │
│  No one renovates without inspecting the existing structure.    │
│  No one occupies a building without a fire escape plan.         │
│                                                                 │
│  Why would software be different?                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 Your Three Audiences

**Every piece of technical communication you write has one of three audiences:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   YOUR THREE AUDIENCES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  AUDIENCE 1: THE REVIEWER / TEAMMATE                            │
│  ────────────────────────────────────                           │
│  Reading your PR right now. Needs to understand:                │
│  • WHAT did you change?                                         │
│  • WHY did you change it?                                       │
│  • Is it CORRECT?                                               │
│                                                                 │
│  They have: Context about the project. Limited time.            │
│  They need: Enough information to make a trust decision.        │
│                                                                 │
│  Format: PR description, review comments, commit messages.      │
│                                                                 │
│                                                                 │
│  AUDIENCE 2: THE FUTURE DEVELOPER                               │
│  ────────────────────────────────                               │
│  Joining the project in 3 months. Needs to understand:          │
│  • WHAT does this system do?                                    │
│  • HOW do I set it up?                                          │
│  • WHERE is the code for feature X?                             │
│                                                                 │
│  They have: Zero context. Fresh eyes.                           │
│  They need: A map. An entry point. A path to understanding.     │
│                                                                 │
│  Format: README, API docs, architecture docs, ADRs.             │
│                                                                 │
│                                                                 │
│  AUDIENCE 3: THE ON-CALL ENGINEER AT 3 AM                       │
│  ────────────────────────────────────────                       │
│  Production is down. Needs to understand:                       │
│  • WHAT is broken?                                              │
│  • HOW do I diagnose it?                                        │
│  • WHAT are the steps to fix it?                                │
│                                                                 │
│  They have: Adrenaline. Pressure. Possibly not fully awake.     │
│  They need: Step-by-step instructions. No ambiguity.            │
│                                                                 │
│  Format: Runbooks, health check endpoints, structured logs.     │
│                                                                 │
│                                                                 │
│  Every document you write serves one of these three.            │
│  If you don't know which audience you're writing for,           │
│  you're writing for nobody.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 2: CODE REVIEW — THE REVIEWER'S CRAFT

## 2.1 The Review Pyramid

**Most developers review backwards. They start at the bottom — the stuff that matters least.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE REVIEW PYRAMID                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Review TOP-DOWN. Spend your time where it matters most.        │
│                                                                 │
│                        /\                                       │
│                       /  \          LEVEL 4: STYLE              │
│                      / 4  \         Formatting, naming          │
│                     /      \        conventions, whitespace     │
│                    /────────\       ➜ Let tools handle this     │
│                   /          \                                   │
│                  /     3      \     LEVEL 3: MAINTAINABILITY    │
│                 /              \    Readability, naming clarity, │
│                /────────────────\   duplication, complexity      │
│               /                  \                               │
│              /        2           \ LEVEL 2: LOGIC              │
│             /                      \ Correctness, edge cases,   │
│            /────────────────────────\ error handling, types     │
│           /                          \                           │
│          /            1               \ LEVEL 1: ARCHITECTURE   │
│         /                              \ Design, security,      │
│        /                                \ patterns, contracts   │
│       /──────────────────────────────────\                      │
│                                                                 │
│                                                                 │
│  ⬆ IMPORTANCE        TIME SPENT BY MOST REVIEWERS ⬇            │
│                                                                 │
│  Most people: "You missed a comma" (Level 4)                    │
│  Good reviewers: "This violates tenant isolation" (Level 1)     │
│                                                                 │
│  ruff handles Level 4.  mypy handles much of Level 2.           │
│  Only HUMANS can review Level 1 effectively.                    │
│  Spend your review time where machines cannot.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Think of it this way:**

A building inspector doesn't start by checking if the paint color matches the permit. They start by checking if the foundation can hold the building up. They check if the fire exits work. They check if the plumbing won't leak into the electrical system. THEN they check the paint.

> "If you spend your entire code review pointing out formatting issues, you have done zero useful work. The linter does that. Your job is to catch the things that linters, type checkers, and tests CANNOT catch. Design decisions. Security holes. Missing edge cases. Architectural violations."

---

## 2.2 What to Catch: Architecture, Logic, Security

**Level 1: Architecture & Design**

These are the most important and hardest-to-fix issues. If you catch them after merge, the cost of change is 10x higher.

```
┌─────────────────────────────────────────────────────────────────┐
│             LEVEL 1: ARCHITECTURE REVIEW                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Ask yourself:                                                  │
│                                                                 │
│  SEPARATION OF CONCERNS                                         │
│  • Does the route handler contain business logic?               │
│    It should delegate to a service. (Week 6: repository pattern)│
│  • Does the service import from FastAPI?                        │
│    It shouldn't — services should be framework-agnostic.        │
│                                                                 │
│  CONTRACTS & BOUNDARIES                                         │
│  • Does the endpoint use a Pydantic request/response model?     │
│    Or raw dicts? (Week 3: Pydantic)                             │
│  • Does adding a new feature require modifying existing code?   │
│    Or just extending it?                                        │
│                                                                 │
│  DATA FLOW                                                      │
│  • Is external data validated before use?                       │
│    (Week 8: "never trust external data")                        │
│  • Is the query in the right layer?                             │
│    Routes should not contain SQL.                               │
│                                                                 │
│  MULTI-TENANCY                                                  │
│  • Can a user from Org A access Org B's resources?              │
│    (Week 13 Lecture 2: tenant isolation)                        │
│  • Is tenant scoping applied at the service/repository level?   │
│    Not just in the route handler?                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Example — a route handler that violates separation of concerns:**

```python
# ❌ ARCHITECTURAL PROBLEM: Business logic embedded in route handler

@router.post("/projects/{project_id}/members", status_code=201)
async def add_member(
    project_id: int,
    payload: AddMemberRequest,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    # Authorization check embedded in route
    project = await db.execute(
        select(Project).where(
            Project.id == project_id,
            Project.org_id == current_user.org_id,
        )
    )
    project = project.scalar_one_or_none()
    if not project:
        raise HTTPException(status_code=404, detail="Project not found")

    # Business rule embedded in route
    member_count = await db.execute(
        select(func.count()).where(ProjectMember.project_id == project_id)
    )
    if member_count.scalar() >= 50:
        raise HTTPException(status_code=400, detail="Member limit reached")

    # Direct DB manipulation in route
    new_member = ProjectMember(
        project_id=project_id,
        user_id=payload.user_id,
        role=payload.role,
    )
    db.add(new_member)
    await db.commit()

    return {"id": new_member.id, "status": "added"}
```

**Why this is an architecture problem, not just "style":**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  The route handler is doing FOUR jobs:                          │
│  ├─ Authorization (checking org_id)                             │
│  ├─ Business rules (member limit of 50)                         │
│  ├─ Data access (raw DB queries)                                │
│  └─ Response formatting (bare dict)                             │
│                                                                 │
│  Consequences:                                                  │
│  ├─ Cannot reuse the "add member" logic elsewhere               │
│  │   (e.g., in a Celery task that bulk-imports members)         │
│  ├─ Cannot test business rules without spinning up FastAPI      │
│  ├─ The "50 member limit" is buried in a route — invisible      │
│  │   to anyone reading service-layer code                       │
│  └─ If the member-limit rule changes, you're editing a route    │
│     handler, not a business logic module                        │
│                                                                 │
│  This is exactly the kind of issue that a linter CANNOT catch   │
│  and a human reviewer MUST catch.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Level 2: Logic & Correctness**

These are bugs waiting to happen. Some mypy catches. Many it does not.

```
┌─────────────────────────────────────────────────────────────────┐
│                LEVEL 2: LOGIC REVIEW                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Ask yourself:                                                  │
│                                                                 │
│  NONE HANDLING                                                  │
│  • Every .scalar_one_or_none() — is the None case handled?     │
│  • Every dict/attribute access — can it be None or missing?     │
│                                                                 │
│  STATUS CODES                                                   │
│  • 404 for not found? (Not 400)                                 │
│  • 201 for creation? (Not 200)                                  │
│  • 204 for delete? (Not 200 with empty body)                    │
│  • 409 for conflict? (e.g., duplicate email)                    │
│                                                                 │
│  EDGE CASES                                                     │
│  • Empty list input? Zero-length string? Negative number?       │
│  • Concurrent requests? What if two people assign the same      │
│    task simultaneously?                                         │
│                                                                 │
│  QUERY CORRECTNESS                                              │
│  • N+1 queries? (Week 6: eager loading)                         │
│  • Missing .where() clause that filters by tenant?              │
│  • Pagination off-by-one?                                       │
│                                                                 │
│  ERROR HANDLING                                                 │
│  • Can an exception leak a stack trace to the client?           │
│  • Are database errors caught and wrapped?                      │
│  • Does a failure leave data in an inconsistent state?          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Example — a subtle None bug:**

```python
# Can you spot the crash?

async def get_task_assignee_email(
    task_id: int,
    db: AsyncSession,
) -> str:
    result = await db.execute(
        select(Task).options(joinedload(Task.assignee)).where(Task.id == task_id)
    )
    task = result.scalar_one_or_none()

    if not task:
        raise TaskNotFoundError(task_id)

    return task.assignee.email  # 💥 What if task.assignee is None?
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  The code checks if TASK is None.                               │
│  It does NOT check if task.ASSIGNEE is None.                    │
│                                                                 │
│  A task can exist but have no assignee.                         │
│  task.assignee → None                                           │
│  None.email → AttributeError in production.                     │
│                                                                 │
│  This is the most common bug pattern in ORM code.               │
│  It passes all tests where tasks HAVE assignees.                │
│  It crashes on the first unassigned task in production.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Level 1+2: Security**

Security issues often span architecture and logic. They deserve special attention because the consequences are outsized.

```
┌─────────────────────────────────────────────────────────────────┐
│                  SECURITY REVIEW                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  For EVERY endpoint, ask:                                       │
│                                                                 │
│  AUTHENTICATION                                                 │
│  • Is get_current_user present? Or is this accidentally public? │
│  • Does the auth dependency actually get used in the logic?     │
│    (It's possible to inject current_user but never CHECK it)    │
│                                                                 │
│  AUTHORIZATION                                                  │
│  • Can a "viewer" role perform a mutation?                      │
│  • Can a member of one org access another org's data?           │
│  • Does the endpoint check BOTH role AND org membership?        │
│                                                                 │
│  DATA EXPOSURE                                                  │
│  • Does the response model exclude sensitive fields?            │
│    (password_hash, internal IDs, tokens)                        │
│  • Do error messages leak implementation details?               │
│    ("Table 'users' has no row with id 42" → information leak)  │
│                                                                 │
│  INPUT                                                          │
│  • Is there a Pydantic model with field constraints?            │
│    Or raw dict/Any that accepts anything?                       │
│  • Are file uploads validated? (size, type, content)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Security bugs don't fail your tests. They don't trigger type errors. They don't cause exceptions in development. They sit quietly until someone exploits them. The code review is your last human checkpoint before production."

---

## 2.3 How to Say It: Feedback That Helps

**The hardest part of code review is not FINDING issues — it's COMMUNICATING them in a way that helps the author learn and improves the code without starting a fight.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE FEEDBACK FRAMEWORK                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Every review comment should have THREE parts:                  │
│                                                                 │
│  1. LABEL      — How important is this?                         │
│  2. OBSERVATION — What did you see? (fact, not judgment)        │
│  3. SUGGESTION  — What would you do instead? (optional if       │
│                   asking a genuine question)                    │
│                                                                 │
│  Format:                                                        │
│  [label] Observation. Suggestion or question.                   │
│                                                                 │
│                                                                 │
│  LABELS:                                                        │
│  ─────────────────────────────────────────────                  │
│  [blocker]    — Must fix before merge. Breaks correctness,      │
│                 security, or data integrity.                    │
│                                                                 │
│  [bug]        — Likely causes incorrect behavior. Should fix.   │
│                                                                 │
│  [suggestion] — Improvement idea. Author decides.               │
│                                                                 │
│  [question]   — I don't understand this. Explain or clarify.    │
│                                                                 │
│  [nit]        — Trivial stylistic preference. Ignore if busy.   │
│                                                                 │
│  [praise]     — This is good. I want to see more of this.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Labels matter enormously.** Without them, every comment feels like a demand. The author can't tell if you're blocking the merge over a missing comma or a security hole.

**Now, the bad versus good examples. All of these refer to the broken `assign_task` endpoint from Section 1.1:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     BAD REVIEW COMMENTS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ "This is wrong."                                            │
│     → What is wrong? What should I do instead?                  │
│                                                                 │
│  ❌ "Use Pydantic."                                             │
│     → Where? Why? How?                                          │
│                                                                 │
│  ❌ "Why didn't you add tests?"                                 │
│     → Accusatory tone. Makes the author defensive.              │
│                                                                 │
│  ❌ "I would never write it this way."                          │
│     → About you, not about the code.                            │
│                                                                 │
│  ❌ "Hmm."                                                      │
│     → Passive-aggressive and useless.                           │
│                                                                 │
│  ❌ "LGTM" (on code with 11 issues)                             │
│     → Rubber-stamping. Worse than no review at all —            │
│       it creates false confidence.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The same issues, communicated well:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     GOOD REVIEW COMMENTS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  ON THE MISSING AUTH (Line 1-4):                                │
│  ────────────────────────────────                               │
│                                                                 │
│  "[blocker] This endpoint has no authentication dependency.     │
│   Any unauthenticated request can reassign tasks. We need       │
│   current_user: User = Depends(get_current_user) here, plus     │
│   a check that current_user belongs to the task's organization  │
│   — otherwise this is a cross-tenant access vulnerability."     │
│                                                                 │
│   ✅ Labeled (blocker — author knows severity)                  │
│   ✅ Specific (points to what's missing)                        │
│   ✅ Explains the consequence (cross-tenant vulnerability)      │
│   ✅ Suggests the fix (add dependency, add org check)           │
│                                                                 │
│                                                                 │
│  ON THE NONE BUG (Line 10-12):                                  │
│  ─────────────────────────────                                  │
│                                                                 │
│  "[bug] If data['user_id'] doesn't match any user,              │
│   .scalar_one_or_none() returns None, and then                  │
│   `user.id` on line 12 raises AttributeError.                   │
│   Add a None check and return 404, same pattern you             │
│   used for the task lookup above."                              │
│                                                                 │
│   ✅ Walks through the exact failure path                       │
│   ✅ References existing code as a model for the fix            │
│                                                                 │
│                                                                 │
│  ON THE RAW DICT (Line 3):                                      │
│  ─────────────────────────                                      │
│                                                                 │
│  "[suggestion] `data: dict` bypasses all Pydantic               │
│   validation. If the request body has no 'user_id' key,         │
│   this raises KeyError — which becomes a 500 instead            │
│   of a clean 422. A request model like                          │
│   `class TaskAssignRequest(BaseModel): user_id: int`            │
│   gives you automatic validation and Swagger docs."             │
│                                                                 │
│   ✅ Explains the concrete failure scenario                     │
│   ✅ Provides exact code for the fix                            │
│   ✅ Names the benefit (validation + docs)                      │
│                                                                 │
│                                                                 │
│  ON THE STATUS CODE (Line 8):                                   │
│  ─────────────────────────                                      │
│                                                                 │
│  "[bug] status_code=400 for 'not found' — 400 means             │
│   bad request (malformed input), 404 means resource             │
│   not found. Clients and monitoring tools rely on               │
│   status codes for categorization."                             │
│                                                                 │
│                                                                 │
│  AND SOMETHING POSITIVE:                                        │
│  ───────────────────────                                        │
│                                                                 │
│  "[praise] Good use of scalar_one_or_none() instead of          │
│   scalar_one() — the explicit None handling pattern             │
│   is exactly right for lookups that might not match."           │
│                                                                 │
│   ✅ Reinforces what they did well                              │
│   ✅ Makes the review feel balanced, not just a list of sins    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The principles behind good comments:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE GOLDEN RULES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. REVIEW THE CODE, NOT THE CODER.                             │
│     ❌ "You forgot to add auth"                                 │
│     ✅ "This endpoint is missing auth"                          │
│     (Small wording change. Big psychological difference.)       │
│                                                                 │
│  2. ASK QUESTIONS WHEN GENUINELY UNCERTAIN.                     │
│     ❌ "This is the wrong pattern" (maybe they know something   │
│        you don't)                                               │
│     ✅ "[question] I see this queries the DB directly instead   │
│        of going through the service layer — was there a         │
│        reason, or should we refactor?"                          │
│                                                                 │
│  3. EXPLAIN THE "WHY," NOT JUST THE "WHAT."                     │
│     ❌ "Use 404 here"                                           │
│     ✅ "Use 404 — monitoring tools group 400s as client input   │
│        errors and 404s as missing resources. Wrong status code  │
│        corrupts our error metrics."                             │
│                                                                 │
│  4. PRAISE WHAT'S GOOD. EXPLICITLY.                             │
│     It's not flattery — it's SIGNAL. It tells the author        │
│     "do more of this." Otherwise they only learn what NOT       │
│     to do, never what TO do.                                    │
│                                                                 │
│  5. ONE COMMENT PER ISSUE. DON'T PILE UP.                       │
│     If the same mistake appears 5 times, comment on the FIRST   │
│     instance and say "same pattern in lines X, Y, Z." Don't    │
│     leave 5 identical comments — it feels like an attack.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 A Full Review Walkthrough

**Let's walk through a complete review as you'd actually do it in your capstone. You open a PR. Here is the process — top-down, systematic.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  REVIEW PROCESS (STEP BY STEP)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 0: READ THE PR DESCRIPTION FIRST (2 min)                  │
│  ─────────────────────────────────────────────                  │
│  Before reading ANY code. Understand: What is the goal?         │
│  What is the approach? What should I expect to see?             │
│  No description? Ask the author to add one before you review.   │
│                                                                 │
│                                                                 │
│  STEP 1: SCAN THE FILE LIST (1 min)                             │
│  ──────────────────────────────────                             │
│  • How many files changed? (>15 files = probably too big)       │
│  • Which layers are touched? (routes? services? models?         │
│    migrations?)                                                 │
│  • Are there tests? (No tests = request them immediately)       │
│  • Any unexpected files? (Why is conftest.py modified?)         │
│                                                                 │
│                                                                 │
│  STEP 2: READ THE MIGRATION FIRST, IF ANY (3 min)               │
│  ─────────────────────────────────────────────────              │
│  The Alembic migration tells you the DATA STORY.                │
│  New table? New column? Index? This frames everything else.     │
│  Check: Is the migration reversible? Is there a downgrade?      │
│                                                                 │
│                                                                 │
│  STEP 3: READ THE MODELS / SCHEMAS (3 min)                      │
│  ──────────────────────────────────────────                     │
│  Pydantic schemas = the API contract.                           │
│  SQLAlchemy models = the data structure.                        │
│  If these are wrong, everything built on them is wrong.         │
│                                                                 │
│                                                                 │
│  STEP 4: READ THE SERVICE LAYER (5 min)                         │
│  ──────────────────────────────────────                         │
│  This is where business logic lives.                            │
│  Check: correctness, authorization, edge cases, tenant          │
│  isolation, error handling.                                     │
│                                                                 │
│                                                                 │
│  STEP 5: READ THE ROUTES (3 min)                                │
│  ────────────────────────────────                               │
│  Routes should be THIN. If they're not, that's a flag.          │
│  Check: status codes, response models, dependency injection,    │
│  Pydantic validation.                                           │
│                                                                 │
│                                                                 │
│  STEP 6: READ THE TESTS (5 min)                                 │
│  ──────────────────────────────                                 │
│  Tests reveal what the author THINKS the code does.             │
│  Check: Are edge cases covered? Are errors tested?              │
│  Is the happy path too happy? (Only tests with valid data)      │
│                                                                 │
│                                                                 │
│  STEP 7: WRITE YOUR COMMENTS (5 min)                            │
│  ────────────────────────────────────                           │
│  Label everything. Lead with blockers. End with praise.         │
│                                                                 │
│                                                                 │
│  TOTAL: ~25 minutes for a well-scoped PR.                       │
│  If it takes longer than 30 min, the PR is too big.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice the order: description → file list → migration → models → services → routes → tests. You read OUTWARD from the data, not inward from the HTTP layer. The data model is the foundation of the building. If the foundation is cracked, it doesn't matter how pretty the walls are."

---

# PART 3: CODE REVIEW — THE AUTHOR'S CRAFT

## 3.1 Writing Reviewable PRs

**The quality of the review is bounded by the quality of the PR.**

A massive, unfocused PR gets a rubber-stamp "LGTM." A small, clear PR gets a thorough, useful review. You control this.

```
┌─────────────────────────────────────────────────────────────────┐
│                   PR SIZE AND REVIEW QUALITY                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  Lines Changed     Review Quality     What Actually Happens     │
│  ──────────────    ──────────────     ──────────────────────    │
│                                                                 │
│  1–50 lines        Deep, thorough     Reviewer reads every      │
│                                       line. Catches subtle bugs.│
│                                                                 │
│  50–200 lines      Good               Reviewer engages but      │
│                                       might skim some sections. │
│                                                                 │
│  200–500 lines     Shallow            Reviewer focuses on the   │
│                                       parts they understand.    │
│                                       Misses interactions.      │
│                                                                 │
│  500+ lines        Rubber stamp       Reviewer scrolls.         │
│                                       Types "LGTM."             │
│                                       Goes back to their work.  │
│                                                                 │
│                                                                 │
│  10 lines of code → 10 issues found.                            │
│  500 lines of code → "looks fine to me."                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The rule: ONE PR, ONE PURPOSE.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  ONE PR, ONE PURPOSE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ BAD PR: "Add task assignment, fix pagination bug,           │
│             refactor user model, update deps"                   │
│                                                                 │
│     → 4 unrelated changes. If task assignment has a bug,        │
│       do you revert the pagination fix too?                     │
│                                                                 │
│                                                                 │
│  ✅ GOOD: Split into 4 PRs:                                     │
│     PR #1: "fix: correct off-by-one in cursor pagination"       │
│     PR #2: "refactor: extract UserProfile from User model"      │
│     PR #3: "chore: update dependencies (monthly)"               │
│     PR #4: "feat: add task assignment endpoint"                  │
│                                                                 │
│     → Each reviewable in isolation.                             │
│     → Each revertible in isolation.                             │
│     → Each has a clear scope and a clear "done."                │
│                                                                 │
│                                                                 │
│  HOW TO SPLIT:                                                  │
│  ├─ Refactor FIRST (separate PR), then build feature ON TOP     │
│  ├─ Infrastructure (migration, model) in one PR,                │
│  │   business logic (service, routes) in the next               │
│  ├─ Bug fixes ALWAYS separate from feature work                 │
│  └─ Dependency updates ALWAYS in their own PR                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 The PR Description

**The PR description is your building permit application. It tells the reviewer: what you're building, why you're building it, and how you built it.**

**A bad PR description:**

```
Title:  updates
Body:   (empty)
```

> "This is the equivalent of handing a building inspector a key and saying 'go look.' They have to reverse-engineer your intentions from the code. They will either give up or miss critical issues."

**A good PR description for your capstone:**

```markdown
## feat(audit): add audit logging for task mutations

### What
Adds audit log entries whenever a task is created, updated,
assigned, or deleted. Entries are stored in a new `audit_log`
table and exposed via a read-only API endpoint.

### Why
Capstone requirement: "Audit log for all mutations."
See ADR-007 for the design decision on row-level audit
vs event-sourcing approach.

### How
- New `AuditLog` SQLAlchemy model with polymorphic `action` enum
- `AuditService.log()` called from `TaskService` after each
  mutation (not in the route handler — keeps separation clean)
- Background task writes the log entry to avoid adding latency
  to the main request path
- New GET /api/v1/audit endpoint (admin-only, paginated)

### Schema Change
New migration: `20260210_add_audit_log_table`
- Creates `audit_log` table (id, action, entity_type,
  entity_id, actor_id, org_id, changes_json, created_at)
- Index on (org_id, created_at) for tenant-scoped queries
- GIN index on changes_json for future JSONB queries

### Testing
- Unit tests: AuditService.log() with all action types
- Integration tests: verify audit entries after task CRUD
- Verified tenant isolation: Org A cannot read Org B's audit log
- 14 new tests, all passing. Coverage: 84% → 86%.

### Notes for Reviewer
- I chose to store `changes_json` as JSONB rather than
  separate `old_value`/`new_value` columns — see ADR-007
  for rationale. Open to discussion.
- The audit write is a BackgroundTask, not a Celery task,
  because the latency is <5ms and we don't need retry logic.
```

**Why each section matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                PR DESCRIPTION ANATOMY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT    → Reviewer knows the scope. Can check: "Does the      │
│            code actually do what the author says it does?"      │
│                                                                 │
│  WHY     → Reviewer knows the motivation. Can judge: "Is       │
│            this the right approach to the right problem?"       │
│            Links to ADR grounds the design.                     │
│                                                                 │
│  HOW     → Reviewer gets a map before reading code. They        │
│            know which files to focus on and in what order.      │
│                                                                 │
│  SCHEMA  → Reviewer reads the migration FIRST (Step 2 of       │
│            the review process). Knowing the index choices       │
│            up front saves time.                                 │
│                                                                 │
│  TESTING → Reviewer knows tests exist and what they cover.      │
│            Can check: "Are there gaps?"                         │
│                                                                 │
│  NOTES   → Pre-answers the questions you KNOW the reviewer     │
│            will ask. Saves a round-trip of comments.            │
│                                                                 │
│                                                                 │
│  A GOOD PR DESCRIPTION SAVES MORE TIME THAN IT TAKES TO WRITE. │
│                                                                 │
│  Writing it: 10 minutes.                                        │
│  Time saved: reviewer doesn't have to guess, comment,           │
│  wait for your reply, re-review. Easily 30+ minutes saved.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Commit History as Narrative (Week 1 Connection)

> "You learned conventional commits in Week 1. Now you'll understand WHY."

Your commit history tells the STORY of how you built the feature. A reviewer reads it to understand your thought process. A future developer reads it to understand how the code evolved. `git blame` uses it to explain why a line exists.

**Bad commit history:**

```
a1b2c3d  wip
d4e5f6a  more stuff
7g8h9i0  fix
b1c2d3e  oops
e4f5g6h  done
f7g8h9i  actually done
j1k2l3m  ok now its done for real
```

> "This tells you nothing. You cannot understand what changed, why it changed, or in what order. If something breaks, you cannot identify which commit introduced the bug. This history is pure noise."

**Good commit history (same feature):**

```
a1b2c3d  feat(audit): add AuditLog model and migration
d4e5f6a  feat(audit): implement AuditService.log()
7g8h9i0  feat(audit): integrate audit logging into TaskService
b1c2d3e  feat(audit): add GET /audit endpoint (admin-only)
e4f5g6h  test(audit): add unit tests for AuditService
f7g8h9i  test(audit): add integration tests for audit endpoint
j1k2l3m  feat(audit): add GIN index on changes_json column
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Each commit:                                                   │
│  ├─ Does ONE thing                                              │
│  ├─ Has a clear conventional commit prefix (feat, test, fix)    │
│  ├─ Has a scope (audit)                                         │
│  ├─ Describes WHAT, not HOW                                     │
│  └─ Builds on the previous — tells a story when read top-down   │
│                                                                 │
│  Now a reviewer can:                                            │
│  ├─ Review commit-by-commit (smaller chunks, higher quality)    │
│  ├─ Understand the build order (model → service → route → test) │
│  └─ git bisect to find which commit introduced a bug            │
│                                                                 │
│  Future developer can:                                          │
│  ├─ git blame a line and see "feat(audit): add GIN index"       │
│  └─ Instantly understand WHY that index exists                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical tip: interactive rebase**

Your WORKING history will be messy. That's fine. Before opening a PR, clean it up:

```bash
# Squash and reorder your last 8 commits into a clean story
git rebase -i HEAD~8
```

This is not about lying. It's about editing your draft before publishing. You don't submit a first draft of an essay. Don't submit a first draft of your commit history.

---

## 3.4 Receiving Feedback Without Ego

**The other side of code review: someone tells you your code has problems.**

This is hard. You spent hours writing it. It feels personal. It is not personal. Here is the framework for responding:

```
┌─────────────────────────────────────────────────────────────────┐
│              RECEIVING FEEDBACK: THE FRAMEWORK                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  STEP 1: ASSUME GOOD INTENT                                     │
│  ────────────────────────────                                   │
│  The reviewer wants the code to be better.                      │
│  They are not attacking you. They are attacking a bug.          │
│  Read every comment as: "Hey, I think there might be            │
│  an issue here" — even if the wording is clumsy.                │
│                                                                 │
│                                                                 │
│  STEP 2: CLASSIFY THE COMMENT                                   │
│  ─────────────────────────────                                  │
│  • Correct and I missed it → Fix it. Say "Good catch, fixed."  │
│  • Correct but I disagree on approach → Explain your reasoning.│
│    "I considered X but went with Y because [reason]. Thoughts?" │
│  • Incorrect — reviewer misread the code → Explain kindly.     │
│    "This is actually handled on line 42 — I should add a       │
│    comment to make it clearer."                                 │
│  • Style preference → If not in a style guide, defer to the    │
│    reviewer OR discuss briefly. Don't die on this hill.         │
│                                                                 │
│                                                                 │
│  STEP 3: RESPOND TO EVERY COMMENT                               │
│  ─────────────────────────────────                              │
│  Even if it's just "Done" or "Fixed." Unresolved comments      │
│  are ambiguous — did you see it? Do you agree? Did you fix it? │
│  Silence is the worst response.                                 │
│                                                                 │
│                                                                 │
│  STEP 4: DON'T TAKE "REQUEST CHANGES" PERSONALLY               │
│  ─────────────────────────────────────────────────              │
│  A reviewer who requests changes is doing their job.            │
│  A reviewer who approves everything is not.                     │
│  The BEST reviewers are the harshest — they catch the bugs      │
│  that would have woken you up at 3 AM.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When you disagree:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 PRODUCTIVE DISAGREEMENT                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ "No, my way is right."                                      │
│                                                                 │
│  ❌ "I already thought about that." (Then why isn't it in       │
│     the PR description?)                                        │
│                                                                 │
│  ❌ (Silently ignore the comment and merge anyway)               │
│                                                                 │
│                                                                 │
│  ✅ "I went with BackgroundTask instead of Celery here          │
│     because the audit write is <5ms and doesn't need retry.     │
│     Celery adds operational complexity (broker health, worker   │
│     management) for a sub-millisecond write. But if you see     │
│     a scenario where this could fail, I'll reconsider."         │
│                                                                 │
│  → States the decision                                          │
│  → Explains the reasoning with concrete numbers                 │
│  → Acknowledges the reviewer's concern                          │
│  → Leaves the door open                                         │
│                                                                 │
│                                                                 │
│  THE RULE: If you have to explain your reasoning more than      │
│  once in review comments, the code or documentation needs a     │
│  comment. The NEXT reviewer will have the same question.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: TECHNICAL DOCUMENTATION

## 4.1 The Documentation Stack

**Different documents serve different audiences at different times.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE DOCUMENTATION STACK                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LAYER          AUDIENCE          WHEN THEY READ IT             │
│  ────────────   ────────────────  ─────────────────────         │
│                                                                 │
│  Code itself    Future developer  While changing the code       │
│  (naming,       (including you)   They are IN the file.         │
│   structure)                                                    │
│       │                                                         │
│       ▼                                                         │
│  Comments &     Future developer  While reading the code        │
│  Docstrings     (including you)   Explains the "WHY" inline.    │
│       │                                                         │
│       ▼                                                         │
│  README         New team member   FIRST thing they read.        │
│                 Evaluator         "What is this? How do I       │
│                                    run it?"                     │
│       │                                                         │
│       ▼                                                         │
│  API Docs       API consumers     While integrating. "What      │
│  (OpenAPI)      Frontend devs     endpoints exist? What do      │
│                                    they accept/return?"         │
│       │                                                         │
│       ▼                                                         │
│  ADRs           Future architect  "WHY was it built this way?"  │
│  (Wk13 L1)     New team member   Months or years later.        │
│       │                                                         │
│       ▼                                                         │
│  Runbooks       On-call engineer  3 AM. Production is down.     │
│                 Ops team          "What do I DO?"               │
│                                                                 │
│                                                                 │
│  Each layer answers a DIFFERENT question.                       │
│  None of them replace each other.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 README: The Front Door

**The README is the front door of your building. If it's locked, confusing, or missing, no one enters.**

**The 30-Second Test:** A developer who has never seen your project opens the README. Within 30 seconds, can they answer:
1. What does this project do?
2. How do I run it locally?
3. Where do I go next?

If the answer to any of these is "no," the README has failed.

**A README structure for your capstone:**

```markdown
# TaskFlow — Project Management SaaS Backend

One-paragraph description: what it is, who it's for, what 
problem it solves. Not how it works — what it DOES.

## Tech Stack

Python 3.12 · FastAPI · PostgreSQL · Redis · Celery · SQLAlchemy 2.0

(Bullet-free. The reader scans this in 2 seconds to know if
they have the right prerequisites.)

## Quick Start

Three to five commands. Copy-paste and it works.
If it requires more than 5 steps, your setup is too complex.

    git clone ...
    cp .env.example .env
    # fill in your database URL in .env
    python -m alembic upgrade head
    uvicorn app.main:app --reload

## Project Structure

    app/
    ├── main.py              ← FastAPI application entry point
    ├── routers/             ← HTTP endpoint definitions
    ├── services/            ← Business logic
    ├── repositories/        ← Database access layer
    ├── models/              ← SQLAlchemy models
    ├── schemas/             ← Pydantic request/response models
    ├── core/                ← Config, security, dependencies
    └── workers/             ← Celery task definitions

(This is the MAP. It tells a new developer where to look.)

## API Documentation

    Once the server is running:
    Swagger UI: http://localhost:8000/docs
    ReDoc:      http://localhost:8000/redoc

## Development

How to run tests, run migrations, start Celery workers.
The commands a developer uses DAILY.

    pytest                           # run all tests
    pytest --cov=app                 # with coverage
    alembic revision --autogenerate  # create migration
    alembic upgrade head             # apply migrations
    celery -A app.worker worker -l info  # start worker

## Architecture Decisions

Link to the /docs/adrs/ directory.
"For major design decisions, see our Architecture Decision 
Records."

## Environment Variables

Table of required env vars with descriptions.
Not the VALUES — the DESCRIPTIONS.

    DATABASE_URL     PostgreSQL connection string
    REDIS_URL        Redis connection string  
    JWT_SECRET_KEY   Secret for JWT signing (min 32 chars)
    ...
```

**What NOT to put in the README:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 README ANTI-PATTERNS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ The entire API reference (that's what OpenAPI/Swagger does) │
│                                                                 │
│  ❌ Detailed architecture explanation (that's what ADRs do)     │
│                                                                 │
│  ❌ Operational procedures (that's what runbooks do)            │
│                                                                 │
│  ❌ A 200-line setup guide (your setup is too complex — fix it) │
│                                                                 │
│  ❌ Badges for every tool you use (noise, not signal)           │
│                                                                 │
│  ❌ Nothing at all (surprisingly common)                        │
│                                                                 │
│                                                                 │
│  The README is a ROUTING DOCUMENT.                              │
│  It tells you what exists and WHERE TO FIND IT.                 │
│  It does not try to contain everything itself.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Runbooks: The 3 AM Guide

**A runbook is a step-by-step document for diagnosing and resolving a specific operational problem. It is written for an engineer who is tired, stressed, and possibly unfamiliar with this part of the system.**

> "The building analogy: the runbook is the fire evacuation plan posted on the wall. When the alarm goes off, no one is calm enough to figure out the exit. The plan must be clear, specific, and tested in advance."

**What a runbook looks like — for your capstone stack:**

```markdown
# RUNBOOK: Celery Workers Not Processing Tasks

## Symptoms
- Tasks stuck in PENDING state for >5 minutes
- Flower dashboard shows 0 active workers
- API actions that trigger background jobs (email notifications,
  report generation) appear to succeed but side effects never happen
- Logs show "Task [id] sent" but no "Task [id] received"

## Likely Causes
1. Celery worker process crashed or was killed (OOM, unhandled exception)
2. Redis (broker) is unreachable
3. Task is raising an exception before completing (check retry count)
4. Worker is alive but stuck on a long-running synchronous task

## Diagnosis Steps

### Step 1: Check if workers are running
    celery -A app.worker inspect active
    
    Expected: List of active workers with their current tasks.
    If empty or connection refused → workers are down. Go to Resolution A.

### Step 2: Check Redis connectivity
    redis-cli -u $REDIS_URL ping
    
    Expected: PONG
    If connection refused → Redis is down. See RUNBOOK: Redis Unavailable.

### Step 3: Check worker logs
    # If running locally:
    # Check terminal output of celery worker
    
    Look for:
    - "Connection refused" → Redis issue (Step 2)
    - "OOM" or "Killed" → Worker ran out of memory. Resolution B.
    - Repeated exceptions in same task → Poison message. Resolution C.

### Step 4: Check Flower dashboard
    Open http://localhost:5555
    
    - Workers tab → Are workers registered? 
    - Tasks tab → Are tasks in RETRY or FAILURE state?
    - If tasks are in STARTED but never finish → Resolution D.

## Resolution

### A. Workers are down — restart
    celery -A app.worker worker -l info
    
    Verify: Check Flower. Workers should appear within 10 seconds.

### B. Workers killed by OOM — reduce concurrency
    celery -A app.worker worker -l info --concurrency=2
    
    Then investigate which task is consuming excessive memory.

### C. Poison message — task fails repeatedly
    1. Identify the task ID from Flower
    2. Check the exception in the task's traceback
    3. If the task is non-critical: revoke it
       celery -A app.worker control revoke <task_id>
    4. Fix the underlying bug, deploy, then retry

### D. Stuck task — worker blocked on synchronous operation
    1. Check: is the task doing CPU-bound work or calling a
       blocking API without timeout?
    2. Kill and restart the worker (unfinished tasks will be
       re-queued if acks_late=True)
    3. Add timeout: soft_time_limit to the task decorator

## Prevention
- Set soft_time_limit on all tasks (Week 11: Celery timeouts)
- Monitor worker count with health check endpoint
- Set up alerting on task failure rate in Flower
```

**The anatomy of a good runbook:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  RUNBOOK ANATOMY                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SYMPTOMS    — How do you KNOW this is the problem?             │
│                Observable effects, not root causes.             │
│                Written as "you will see..." not "if X then..."  │
│                                                                 │
│  LIKELY      — Ranked by probability. Most common first.        │
│  CAUSES        Saves time — check the likely cause before       │
│                the unlikely one.                                │
│                                                                 │
│  DIAGNOSIS   — Step-by-step. Each step has:                     │
│  STEPS         • The COMMAND to run (copy-pasteable)            │
│                • The EXPECTED output                            │
│                • What to do if the output differs               │
│                • Which resolution to jump to                    │
│                                                                 │
│  RESOLUTION  — Labeled (A, B, C). Each is a specific fix        │
│                for a specific diagnosis.                        │
│                Not "debug the issue" but "run THIS command."    │
│                                                                 │
│  PREVENTION  — How to stop this from happening again.           │
│                Links to monitoring, configuration changes.      │
│                                                                 │
│                                                                 │
│  KEY RULE: Every command in a runbook must be COPY-PASTEABLE.   │
│  No placeholders like <your-connection-string>.                 │
│  Use environment variables: $REDIS_URL, $DATABASE_URL.          │
│  The reader's hands are shaking. Don't make them think.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What runbooks should your capstone have?**

```
┌─────────────────────────────────────────────────────────────────┐
│            RECOMMENDED RUNBOOKS FOR YOUR CAPSTONE               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. API is returning 500 errors                                 │
│     (database connection, unhandled exceptions)                 │
│                                                                 │
│  2. Celery workers not processing tasks                         │
│     (the example above)                                         │
│                                                                 │
│  3. Redis is unreachable                                        │
│     (cache degradation, token storage failure)                  │
│                                                                 │
│  4. Database migrations failed in deployment                    │
│     (Alembic rollback procedures)                               │
│                                                                 │
│  5. WebSocket connections dropping                              │
│     (connection manager, scaling issues)                        │
│                                                                 │
│  You don't need 50 runbooks. You need runbooks for the          │
│  5 things most likely to go wrong in YOUR system.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Inline Documentation: When and How

**Comments and docstrings. The most abused and the most neglected form of documentation, simultaneously.**

**The cardinal rule:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Code tells you WHAT and HOW.                                   │
│  Comments tell you WHY.                                         │
│                                                                 │
│  If a comment restates what the code does, delete it.           │
│  If a comment explains why the code does something              │
│  non-obvious, keep it forever.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Bad comments — delete all of these:**

```python
# ❌ RESTATING THE CODE (the code already says this)

# Get the user from the database
user = await user_repo.get_by_id(user_id)

# Check if user is None
if user is None:
    raise UserNotFoundError(user_id)

# Set the status to active
user.status = UserStatus.ACTIVE

# Increment the counter
counter += 1

# Return the result
return result
```

> "Every one of these comments is noise. They say what the code says. If the code is clear, these comments add nothing. If the code is NOT clear, the solution is to make the code clearer — not to add a comment explaining unclear code."

**Good comments — these explain WHY:**

```python
# ✅ EXPLAINING WHY (the code can't tell you this)

# We use keyset pagination instead of OFFSET because our audit log
# table has 2M+ rows and OFFSET degrades linearly. See ADR-009.
query = select(AuditLog).where(AuditLog.id > cursor).limit(page_size)

# Retry up to 3 times with exponential backoff. The GitHub API
# returns 502 intermittently under load — this is documented:
# https://github.com/github/rest-api-description/issues/XXX
@retry(stop=stop_after_attempt(3), wait=wait_exponential())
async def fetch_github_repos(org: str) -> list[dict]:
    ...

# We intentionally do NOT eagerly load task.comments here because
# this endpoint is called by the dashboard list view, which only
# shows task titles. Loading comments would fetch ~50x more data.
query = select(Task).where(Task.project_id == project_id)

# The 30-second timeout is intentionally shorter than the Celery
# soft_time_limit (60s) so the API returns a timeout error to the
# client BEFORE the worker kills the task. This prevents silent
# failures.
EXTERNAL_API_TIMEOUT = 30
```

**Notice the pattern:** every good comment answers one of these questions:
- "Why this approach instead of the obvious one?"
- "Why this specific value?"
- "What constraint or external factor drove this decision?"
- "What would go wrong if someone changed this?"

**Docstrings — for functions that are called by OTHER people:**

```python
# ❌ USELESS DOCSTRING (restates the signature)

async def get_user(user_id: int) -> User:
    """Get a user by their ID."""
    ...

# The type hints already tell you it takes an int and returns a User.
# The function name already tells you it gets a user.
# This docstring adds ZERO information.


# ✅ USEFUL DOCSTRING (tells you what the types don't)

async def assign_task(
    task_id: int,
    assignee_id: int,
    assigned_by: User,
) -> Task:
    """Assign a task to a team member.

    Validates that the assignee belongs to the same organization
    as the task. Sends a WebSocket notification to the assignee
    and creates an audit log entry.

    Raises:
        TaskNotFoundError: If task_id doesn't match a task in
            the current user's organization.
        UserNotFoundError: If assignee_id doesn't match a user
            in the same organization.
        PermissionError: If assigned_by doesn't have 'admin' or
            'member' role in the task's project.
    """
    ...
```

**When to write a docstring:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  WRITE A DOCSTRING WHEN:                                        │
│  ├─ The function is part of a public interface (service layer,  │
│  │   repository methods) — other modules call it                │
│  ├─ The function has non-obvious SIDE EFFECTS (sends email,     │
│  │   writes audit log, invalidates cache)                       │
│  ├─ The function RAISES exceptions that callers need to handle  │
│  └─ The behavior is more complex than the signature suggests    │
│                                                                 │
│  SKIP THE DOCSTRING WHEN:                                       │
│  ├─ The function is private/internal and self-explanatory       │
│  ├─ The function name + type hints tell the full story          │
│  └─ The function is a thin wrapper (route handler that calls    │
│     a service method — the service has the docstring)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: ESTIMATION & NAVIGATING CODEBASES

## 5.1 Breaking Down Features Into Tasks

**Your capstone requires you to plan your work. In your career, you will be asked "how long will this take?" constantly. The answer is never a single number — it's a decomposition.**

**Example: "Add file attachment support to tasks"**

This is a capstone requirement. How long does it take? If you say "a few days," you're guessing. If you break it down, you're estimating.

```
┌─────────────────────────────────────────────────────────────────┐
│         BREAKING DOWN: FILE ATTACHMENT SUPPORT                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Start with the question: "What layers of the system            │
│  does this feature touch?"                                      │
│                                                                 │
│                                                                 │
│  DATABASE LAYER                                                 │
│  ├─ Design Attachment model                            (1 hr)   │
│  │   (id, task_id, filename, content_type, size,                │
│  │    storage_path, uploaded_by, created_at)                    │
│  └─ Alembic migration                                 (0.5 hr) │
│                                                                 │
│  SCHEMA LAYER                                                   │
│  ├─ AttachmentCreate (Pydantic request model)          (0.5 hr) │
│  └─ AttachmentResponse (Pydantic response model)       (0.5 hr) │
│                                                                 │
│  STORAGE LAYER                                                  │
│  ├─ Storage abstraction (interface for save/get/delete)(1.5 hr) │
│  ├─ Local filesystem implementation (for dev)          (1 hr)   │
│  └─ File type & size validation                        (1 hr)   │
│                                                                 │
│  SERVICE LAYER                                                  │
│  ├─ AttachmentService.upload()                         (1.5 hr) │
│  │   (validate, store, create DB record, audit log)             │
│  ├─ AttachmentService.download()                       (1 hr)   │
│  └─ AttachmentService.delete()                         (1 hr)   │
│      (remove file, delete DB record, audit log)                 │
│                                                                 │
│  ROUTE LAYER                                                    │
│  ├─ POST /tasks/{id}/attachments (multipart upload)    (1.5 hr) │
│  ├─ GET  /tasks/{id}/attachments (list)                (0.5 hr) │
│  ├─ GET  /attachments/{id}/download                    (1 hr)   │
│  └─ DELETE /attachments/{id}                           (0.5 hr) │
│                                                                 │
│  TESTING                                                        │
│  ├─ Unit tests for AttachmentService                   (2 hr)   │
│  ├─ Integration tests for endpoints                    (1.5 hr) │
│  └─ Test file fixtures and cleanup                     (0.5 hr) │
│                                                                 │
│  CROSS-CUTTING CONCERNS (easy to forget!)                       │
│  ├─ Tenant isolation: ensure files scoped to org       (0.5 hr) │
│  ├─ Cleanup: delete files when task is deleted         (1 hr)   │
│  ├─ Audit log entries for upload/delete                (0.5 hr) │
│  └─ Cache invalidation for task detail (has_attachments)(0.5 hr)│
│                                                                 │
│                                                                 │
│  RAW ESTIMATE:   ~18 hours                                      │
│  WITH BUFFER:    ~25-28 hours                                   │
│  (The buffer accounts for: debugging, unexpected edge cases,    │
│   integration issues, interruptions, thinking time)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice the cross-cutting concerns section at the bottom. That's where estimates blow up. The feature SEEMS simple — upload a file, store it, serve it. But tenant isolation, cleanup on delete, audit logging, cache invalidation — these are invisible until you start building. The breakdown FORCES you to see them upfront."

**The decomposition process:**

```
┌─────────────────────────────────────────────────────────────────┐
│                HOW TO DECOMPOSE A FEATURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. LIST THE LAYERS IT TOUCHES                                  │
│     Database? Schemas? Service? Routes? Tests? External API?    │
│     Cache? Background jobs? WebSocket events?                   │
│                                                                 │
│  2. FOR EACH LAYER, LIST INDIVIDUAL CHANGES                     │
│     One task = one thing you can commit and verify.             │
│     If a task takes more than 3 hours, break it down further.  │
│                                                                 │
│  3. IDENTIFY CROSS-CUTTING CONCERNS                             │
│     What existing features need to change?                      │
│     Tenant isolation? Audit logging? Cache invalidation?        │
│     Error handling? Permissions? Background jobs?               │
│                                                                 │
│  4. ADD BUFFER                                                  │
│     Junior devs: multiply raw estimate by 2x                   │
│     Experienced devs: multiply by 1.5x                         │
│     Unfamiliar domain/technology: multiply by 2.5x             │
│                                                                 │
│     This isn't pessimism — it's realism.                        │
│     The buffer covers: debugging, thinking, context-switching,  │
│     code review iterations, and the things you forgot.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Why Estimates Fail (And How to Improve)

```
┌─────────────────────────────────────────────────────────────────┐
│                 WHY ESTIMATES ARE HARD                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THINGS YOU ESTIMATE:                                           │
│  ├─ Writing the code                                            │
│  └─ Writing the tests                                           │
│                                                                 │
│  THINGS YOU FORGET TO ESTIMATE:                                 │
│  ├─ Understanding the existing code you need to change          │
│  ├─ Debugging the thing that "should work but doesn't"          │
│  ├─ Discovering a missing database index during testing         │
│  ├─ Waiting for code review (and doing the revisions)           │
│  ├─ Realizing your Pydantic schema doesn't handle a real-world  │
│  │   edge case you didn't think of                              │
│  ├─ Writing the migration, testing it, testing the rollback     │
│  ├─ Updating the README and API docs                            │
│  └─ The meeting that interrupted your focus for an hour         │
│                                                                 │
│                                                                 │
│  THE WRITING IS 40% OF THE WORK.                                │
│  THE OTHER 60% IS EVERYTHING AROUND IT.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical rule for your capstone:**

> "Estimate each subtask individually. Sum them up. Then add 40-50% buffer. Track your actual time against estimates. After 3-4 features, you'll know your personal multiplier. Some of you will be 1.5x. Some will be 3x. Neither is a problem — but you need to KNOW your number."

---

## 5.3 Reading Code You Didn't Write

**You will spend more time in your career reading code than writing it. In your capstone, you may work with a partner's code. In your first job, you will join an existing codebase with hundreds of thousands of lines. You need a strategy.**

**The wrong approach:**

```
┌─────────────────────────────────────────────────────────────────┐
│               ❌ HOW MOST PEOPLE READ CODE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Open a random file                                          │
│  2. Start reading from line 1                                   │
│  3. Get confused by line 15                                     │
│  4. Jump to another file                                        │
│  5. Get more confused                                           │
│  6. Open 14 tabs                                                │
│  7. Close everything and say "this code is terrible"            │
│                                                                 │
│  This is not reading. This is wandering.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The systematic approach for a FastAPI codebase:**

```
┌─────────────────────────────────────────────────────────────────┐
│               ✅ SYSTEMATIC CODE READING                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: READ THE README (5 min)                                │
│  ────────────────────────────────                               │
│  What does this project do? How is it structured?               │
│  Get the 30-second overview.                                    │
│                                                                 │
│                                                                 │
│  STEP 2: READ THE PROJECT STRUCTURE (5 min)                     │
│  ───────────────────────────────────────────                    │
│  Just the directory tree. Don't open any file yet.              │
│  Understand the SHAPE. Where are routes? Services? Models?      │
│                                                                 │
│     app/                                                        │
│     ├── main.py           ← Start here. Always.                │
│     ├── routers/          ← HTTP layer                         │
│     ├── services/         ← Business logic                     │
│     ├── repositories/     ← Database access                    │
│     ├── models/           ← SQLAlchemy models                  │
│     └── schemas/          ← Pydantic models                    │
│                                                                 │
│                                                                 │
│  STEP 3: READ main.py (3 min)                                   │
│  ─────────────────────────────                                  │
│  This is the entry point. It shows you:                         │
│  • What routers are registered (the API surface)                │
│  • What middleware is installed                                 │
│  • What startup/shutdown events exist                           │
│  • How the app is configured                                    │
│                                                                 │
│                                                                 │
│  STEP 4: PICK ONE FEATURE AND TRACE IT END-TO-END (20 min)      │
│  ──────────────────────────────────────────────────────         │
│  Pick a route — say, POST /tasks.                               │
│  Trace the FULL request path:                                   │
│                                                                 │
│     Router (tasks.py)                                           │
│       │ What does the endpoint signature look like?             │
│       │ What dependencies does it inject?                       │
│       ▼                                                         │
│     Schema (task_schemas.py)                                    │
│       │ What does the request body look like?                   │
│       │ What does the response look like?                       │
│       ▼                                                         │
│     Service (task_service.py)                                   │
│       │ What business logic runs?                               │
│       │ What can go wrong? What exceptions?                     │
│       ▼                                                         │
│     Repository (task_repository.py)                             │
│       │ What query runs?                                        │
│       │ What does the database interaction look like?           │
│       ▼                                                         │
│     Model (task.py)                                             │
│       │ What columns exist? What relationships?                 │
│       ▼                                                         │
│     Test (test_tasks.py)                                        │
│       What does the test tell you about expected behavior?      │
│                                                                 │
│  By tracing ONE feature, you understand the ENTIRE pattern.     │
│  Every other feature follows the same flow.                     │
│                                                                 │
│                                                                 │
│  STEP 5: READ THE TESTS (15 min)                                │
│  ────────────────────────────────                               │
│  Tests are EXECUTABLE DOCUMENTATION. They show you:             │
│  • What inputs are valid                                        │
│  • What outputs are expected                                    │
│  • What edge cases exist                                        │
│  • What errors the code handles                                 │
│                                                                 │
│  Read the test names first. They tell you the spec:             │
│  • test_assign_task_success                                     │
│  • test_assign_task_not_found_returns_404                       │
│  • test_assign_task_cross_tenant_returns_403                    │
│  • test_assign_task_viewer_role_returns_403                     │
│                                                                 │
│  Those four test names tell you more about the feature          │
│  than 50 lines of comments would.                               │
│                                                                 │
│                                                                 │
│  STEP 6: READ THE ADRs AND MIGRATIONS (10 min)                  │
│  ──────────────────────────────────────────────                 │
│  ADRs explain WHY the architecture looks this way.              │
│  Migrations show HOW the schema evolved over time.              │
│  Together, they give you the HISTORY of the codebase.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "After Steps 1-6, you don't understand the ENTIRE codebase. You understand ONE path through it, deeply. That's enough. You understand the pattern. Now every other feature is a variation of the pattern you already traced."

---

## 5.4 Your First Contribution to an Existing Codebase

```
┌─────────────────────────────────────────────────────────────────┐
│              YOUR FIRST PR IN A NEW CODEBASE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ WRONG FIRST PR:                                             │
│     "I refactored the entire authentication system              │
│      because I think it could be better."                       │
│                                                                 │
│     → You don't understand the codebase yet.                    │
│     → You don't know WHY the current design exists.             │
│     → You're changing load-bearing walls before you've          │
│       understood the floor plan.                                │
│                                                                 │
│                                                                 │
│  ✅ GOOD FIRST PRs:                                             │
│                                                                 │
│  • Fix a small bug (forces you to trace one code path)          │
│                                                                 │
│  • Add a missing test (forces you to understand expected        │
│    behavior)                                                    │
│                                                                 │
│  • Improve error handling (forces you to understand what        │
│    can go wrong)                                                │
│                                                                 │
│  • Add a docstring or comment (forces you to understand         │
│    the code well enough to explain it)                          │
│                                                                 │
│  • Fix a type hint (forces you to understand data flow)         │
│                                                                 │
│                                                                 │
│  Each of these is SMALL, LOW-RISK, and HIGH-LEARNING.           │
│  You build context by touching the codebase gently,             │
│  not by ripping it apart.                                       │
│                                                                 │
│                                                                 │
│  The building analogy: before you knock down a wall,            │
│  you check if it's load-bearing. You do this by studying        │
│  the building — not by swinging the sledgehammer and            │
│  seeing what collapses.                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Card: Code Review

```
┌─────────────────────────────────────────────────────────────────┐
│                 CODE REVIEW QUICK REFERENCE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REVIEW ORDER:                                                  │
│  PR description → File list → Migration → Models →              │
│  Schemas → Services → Routes → Tests                            │
│                                                                 │
│  COMMENT LABELS:                                                │
│      [blocker]    Must fix. Security, correctness, data loss.   │
│      [bug]        Likely incorrect behavior. Should fix.        │
│      [suggestion] Improvement idea. Author decides.             │
│      [question]   I don't understand. Explain or clarify.       │
│      [nit]        Trivial. Ignore if busy.                      │
│      [praise]     This is good. Do more of this.                │
│                                                                 │
│  COMMENT FORMAT:                                                │
│      [label] Observation. Suggestion/question.                  │
│                                                                 │
│  REVIEW CHECKLIST:                                              │
│      □ Auth present on mutating endpoints?                      │
│      □ Tenant isolation enforced?                               │
│      □ Pydantic models for request/response?                    │
│      □ None cases handled after DB queries?                     │
│      □ Correct HTTP status codes?                               │
│      □ Business logic in service layer, not route?              │
│      □ N+1 queries avoided? (eager loading where needed)        │
│      □ Tests cover happy path AND error cases?                  │
│      □ Migration is reversible?                                 │
│      □ No secrets or credentials in code?                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Quick Reference Card: PR Description

```
┌─────────────────────────────────────────────────────────────────┐
│               PR DESCRIPTION TEMPLATE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TITLE:  type(scope): concise description                       │
│          feat(tasks): add task assignment endpoint               │
│                                                                 │
│  ## What                                                        │
│  What does this PR do? (2-3 sentences)                          │
│                                                                 │
│  ## Why                                                         │
│  What problem does it solve? Link to ADR if relevant.           │
│                                                                 │
│  ## How                                                         │
│  Technical approach. Which layers changed and why.              │
│                                                                 │
│  ## Schema Changes (if any)                                     │
│  New tables, columns, indexes, migration name.                  │
│                                                                 │
│  ## Testing                                                     │
│  What tests were added? What was manually verified?             │
│                                                                 │
│  ## Notes for Reviewer (if any)                                 │
│  Design decisions you want feedback on. Known limitations.      │
│  Pre-answer the questions you know they'll ask.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

# Quick Reference Card: Inline Documentation

```
┌─────────────────────────────────────────────────────────────────┐
│             WHEN TO WRITE COMMENTS & DOCSTRINGS                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WRITE A COMMENT WHEN:                                          │
│      The code does something non-obvious.                       │
│      There's a performance reason for the approach.             │
│      There's an external constraint (API quirk, bug workaround).│
│      A specific value has a reason (timeout = 30, not 60).      │
│                                                                 │
│  DELETE A COMMENT WHEN:                                         │
│      It restates what the code does.                            │
│      The code could be renamed to make the comment unnecessary. │
│      It's stale (describes code that has since changed).        │
│                                                                 │
│  WRITE A DOCSTRING WHEN:                                        │
│      The function is called by other modules.                   │
│      The function has non-obvious side effects.                 │
│      The function raises exceptions callers must handle.        │
│      The behavior is more complex than the signature suggests.  │
│                                                                 │
│  FORMAT:                                                        │
│      Comments → explain WHY, not WHAT                           │
│      Docstrings → explain BEHAVIOR, SIDE EFFECTS, ERRORS       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  CODE IS COMMUNICATION                                          │
│                                                                 │
│  You are not writing code for the machine.                      │
│  You are writing code for the next human.                       │
│  That human might be your teammate. Your reviewer.              │
│  The new hire. The on-call engineer. Future you.                │
│                                                                 │
│  Everything in this lecture is the same idea                    │
│  applied to different channels:                                 │
│                                                                 │
│  ┌─────────────────┐     ┌─────────────────────────────┐       │
│  │  CODE REVIEW     │────▶│ Communicating about code     │       │
│  │  comments        │     │ with your reviewer/author    │       │
│  └─────────────────┘     └─────────────────────────────┘       │
│  ┌─────────────────┐     ┌─────────────────────────────┐       │
│  │  PR DESCRIPTION  │────▶│ Communicating your intent    │       │
│  │                  │     │ to whoever reads the change  │       │
│  └─────────────────┘     └─────────────────────────────┘       │
│  ┌─────────────────┐     ┌─────────────────────────────┐       │
│  │  README          │────▶│ Communicating with someone   │       │
│  │                  │     │ who has never seen the code  │       │
│  └─────────────────┘     └─────────────────────────────┘       │
│  ┌─────────────────┐     ┌─────────────────────────────┐       │
│  │  RUNBOOK         │────▶│ Communicating with someone   │       │
│  │                  │     │ under pressure at 3 AM       │       │
│  └─────────────────┘     └─────────────────────────────┘       │
│  ┌─────────────────┐     ┌─────────────────────────────┐       │
│  │  ESTIMATE        │────▶│ Communicating complexity      │       │
│  │                  │     │ to your team / your manager  │       │
│  └─────────────────┘     └─────────────────────────────┘       │
│  ┌─────────────────┐     ┌─────────────────────────────┐       │
│  │  COMMENTS &      │────▶│ Communicating with the next  │       │
│  │  DOCSTRINGS      │     │ developer who opens the file │       │
│  └─────────────────┘     └─────────────────────────────┘       │
│                                                                 │
│                                                                 │
│  THE BUILDING ANALOGY:                                          │
│  ├─ Code Review    = Building inspection                        │
│  ├─ PR Description = Building permit application                │
│  ├─ README         = Welcome guide at the front door            │
│  ├─ ADR            = Architectural blueprint (Week 13 Lecture 1)│
│  ├─ Runbook        = Fire evacuation plan on the wall           │
│  ├─ Estimation     = Construction project schedule              │
│  └─ Reading code   = Studying the building before renovating    │
│                                                                 │
│  Every one of these serves a DIFFERENT audience                 │
│  at a DIFFERENT time with a DIFFERENT need.                     │
│  Write for the audience. Not for yourself.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Upcoming Work

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOUR CAPSTONE (This week and next):                            │
│  └─ Everything in this lecture applies IMMEDIATELY.             │
│     Your capstone requires:                                     │
│     ├─ Clean git history with conventional commits              │
│     ├─ Architecture Decision Records                            │
│     ├─ API documentation via OpenAPI                            │
│     ├─ Comprehensive test suite                                 │
│     └─ README with architecture diagram                         │
│     This lecture is the HOW for those requirements.             │
│                                                                 │
│  WEEK 15 (CI/CD):                                               │
│  └─ The Level 4 review items (formatting, style) will be        │
│     automated by your CI pipeline — ruff for linting, mypy      │
│     for type checking, pytest for correctness. CI removes the   │
│     lowest-value review work so humans focus on what matters.   │
│                                                                 │
│  WEEK 16 (Career Prep):                                         │
│  └─ Technical communication is evaluated in EVERY interview.    │
│     How you talk about your capstone. How you explain a         │
│     design decision. How you write code in a live session.      │
│     Your PR history IS your portfolio. Your README IS your      │
│     first impression. This lecture is career preparation        │
│     disguised as project work.                                  │
│                                                                 │
│  YOUR FIRST JOB:                                                │
│  └─ On Day 1, you will open a codebase you've never seen.      │
│     Step 5.3 of this lecture is literally your Day 1 playbook.  │
│     On Day 7, you will submit your first PR.                    │
│     Section 3 of this lecture is how you make it a good one.    │
│     The engineers who advance fastest are the ones who          │
│     communicate clearly — in code, in reviews, in docs.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```