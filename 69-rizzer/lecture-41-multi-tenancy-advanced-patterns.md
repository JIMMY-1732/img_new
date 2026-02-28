# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CATASTROPHE FIRST, ARCHITECTURE SECOND                         │
│  ──────────────────────────────────────                         │
│  Students must SEE a cross-tenant data leak before learning     │
│  how to prevent one. Fear is a great teacher when the stakes    │
│  are real.                                                      │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Multi-tenancy is abstract infrastructure thinking.             │
│  We use an apartment building analogy throughout.               │
│  Every strategy maps to a physical architecture.                │
│                                                                 │
│  BUILD ON PRIOR TOOLS, DON'T RE-TEACH THEM                     │
│  ──────────────────────────────────────────                     │
│  SQLAlchemy models → just extend them with a mixin              │
│  Depends() chain → add one more link for tenant context         │
│  Repository pattern → scope it, don't rewrite it                │
│  JSONB columns → store audit diffs                              │
│  Celery tasks → schedule hard deletes                           │
│                                                                 │
│  EVERY QUERY, EVERY TIME                                        │
│  ───────────────────────                                        │
│  The central mantra of this lecture. Repeat it until it hurts.  │
│  Multi-tenancy is not a feature you bolt on. It's a discipline  │
│  that touches every line of data access code.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│              MULTI-TENANCY & ADVANCED PATTERNS                  │
│                     (3.5-4 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE CATASTROPHE (35 min)                               │
│  ├─ 1.1 The Data Leak (Demonstration)                           │
│  ├─ 1.2 What is Multi-Tenancy?                                  │
│  ├─ 1.3 The Apartment Building Analogy                          │
│  └─ 1.4 The Core Challenge: Every Query, Every Time             │
│                                                                 │
│  PART 2: MULTI-TENANCY STRATEGIES (40 min)                      │
│  ├─ 2.1 The Isolation Spectrum                                  │
│  ├─ 2.2 Database-per-Tenant                                     │
│  ├─ 2.3 Schema-per-Tenant                                       │
│  ├─ 2.4 Row-Level Isolation (Shared Everything)                 │
│  └─ 2.5 The Decision Framework                                  │
│                                                                 │
│  PART 3: IMPLEMENTING ROW-LEVEL ISOLATION (60 min)              │
│  ├─ 3.1 The Foundation: Organizations & Memberships             │
│  ├─ 3.2 The Tenant Column (org_id Everywhere)                   │
│  ├─ 3.3 Tenant-Aware Model Mixin                                │
│  ├─ 3.4 Resolving Tenant Context (The Dependency Chain)         │
│  ├─ 3.5 Tenant-Scoped Repositories                              │
│  └─ 3.6 The Horror: One Missing WHERE Clause                    │
│                                                                 │
│  PART 4: SHARED VS TENANT-SPECIFIC RESOURCES (20 min)           │
│  ├─ 4.1 Drawing the Line                                        │
│  ├─ 4.2 Classification Examples                                 │
│  └─ 4.3 Modeling the Boundary in Code                           │
│                                                                 │
│  PART 5: AUDIT LOGGING (40 min)                                 │
│  ├─ 5.1 The "Who Changed This?" Problem                         │
│  ├─ 5.2 Anatomy of an Audit Event                               │
│  ├─ 5.3 The Audit Log Table                                     │
│  ├─ 5.4 Recording Changes (Service Layer)                       │
│  ├─ 5.5 Async Audit Logging (Background Processing)             │
│  └─ 5.6 Querying the Audit Trail                                │
│                                                                 │
│  PART 6: SOFT DELETES & DATA RETENTION (30 min)                 │
│  ├─ 6.1 Why DELETE is Dangerous                                 │
│  ├─ 6.2 The Soft Delete Mixin                                   │
│  ├─ 6.3 Filtering by Default                                    │
│  ├─ 6.4 Cascading Soft Deletes                                  │
│  └─ 6.5 Hard Delete Policies (Scheduled Cleanup)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE CATASTROPHE

## 1.1 The Data Leak

**Start with a disaster. Make it feel real.**

> "Your Task Manager SaaS is live. Two companies signed up yesterday: Acme Corp and Wayne Enterprises. This morning, a Wayne Enterprises manager logs in and sees this:"

```python
# This is your existing route — looks fine, right?
# (Simplified from what you've built over the past 12 weeks)

@router.get("/projects", response_model=list[ProjectResponse])
async def list_projects(
    session: AsyncSession = Depends(get_session),
    current_user: User = Depends(get_current_user),  # JWT auth — Week 9
):
    stmt = select(Project)
    result = await session.execute(stmt)
    return result.scalars().all()
```

**Here's what's in the database:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    projects TABLE                               │
├──────┬──────────────────────────────┬───────────────────────────┤
│  id  │  name                        │  created_by               │
├──────┼──────────────────────────────┼───────────────────────────┤
│  1   │  "Website Redesign"          │  alice@acme.com           │
│  2   │  "Q4 Earnings Report"        │  bob@acme.com             │
│  3   │  "Project Gotham"            │  bruce@wayne.com          │ 
│  4   │  "Arkham Security Upgrade"   │  lucius@wayne.com         │
│  5   │  "Merger Due Diligence"      │  alice@acme.com           │
└──────┴──────────────────────────────┴───────────────────────────┘
```

**What does `SELECT * FROM projects` return?**

**Everything. All five rows. To everyone.**

```
┌─────────────────────────────────────────────────────────────────┐
│            WAYNE ENTERPRISES DASHBOARD                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your Projects:                                                 │
│  ├─ Project Gotham                                              │
│  ├─ Arkham Security Upgrade                                     │
│  ├─ Website Redesign            ← 🚨 Acme Corp's project!      │
│  ├─ Q4 Earnings Report          ← 🚨 Acme Corp's financials!   │
│  └─ Merger Due Diligence        ← 🚨 Acme Corp's M&A data!     │
│                                                                 │
│  Bruce Wayne can see Acme Corp's secret merger plans.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "Acme Corp is about to acquire a company. Wayne Enterprises now knows about it. That's insider trading risk, regulatory violation, and the end of your SaaS company. One `SELECT *` — no WHERE clause — and you're in court."

> "The route has authentication. Bruce Wayne is logged in. JWT is valid. RBAC says he's an admin. Every security layer you built in Week 9 says 'allow.' But it still returns data that doesn't belong to him. Why?"

Answer: **Authentication tells you WHO the user is. It doesn't tell you WHICH ORGANIZATION'S DATA they should see. Those are two completely different questions.**

---

## 1.2 What is Multi-Tenancy?

**Define the core concept:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     MULTI-TENANCY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TENANT = One customer organization using your SaaS product.    │
│                                                                 │
│  MULTI-TENANCY = Multiple tenants share the same running        │
│  application, but each tenant's data is INVISIBLE to others.    │
│                                                                 │
│  Examples you use every day:                                    │
│  ├─ Slack     → Each workspace is a tenant                      │
│  ├─ GitHub    → Each organization is a tenant                   │
│  ├─ Notion    → Each team workspace is a tenant                 │
│  ├─ Jira      → Each company site is a tenant                   │
│  └─ Your app  → Each organization is a tenant                   │
│                                                                 │
│  In YOUR capstone: Organization = Tenant.                       │
│  Acme Corp is one tenant. Wayne Enterprises is another.         │
│  Same app, same database, same server — completely separate     │
│  data views.                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Single-tenancy vs multi-tenancy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  SINGLE-TENANT (one app per customer):                          │
│                                                                 │
│    Acme Corp         Wayne Enterprises      Stark Industries    │
│    ┌─────────┐       ┌─────────┐            ┌─────────┐        │
│    │  App    │       │  App    │            │  App    │        │
│    │  + DB   │       │  + DB   │            │  + DB   │        │
│    └─────────┘       └─────────┘            └─────────┘        │
│    Server 1          Server 2               Server 3           │
│                                                                 │
│    3 customers = 3 deployments = 3x cost = 3x maintenance      │
│                                                                 │
│                                                                 │
│  MULTI-TENANT (one app, all customers):                         │
│                                                                 │
│    ┌──────────────────────────────────┐                         │
│    │         One Application          │                         │
│    │  ┌──────┐  ┌──────┐  ┌────────┐ │                         │
│    │  │ Acme │  │Wayne │  │ Stark  │ │                         │
│    │  │ data │  │ data │  │  data  │ │                         │
│    │  └──────┘  └──────┘  └────────┘ │                         │
│    │            One Database          │                         │
│    └──────────────────────────────────┘                         │
│    Server 1                                                     │
│                                                                 │
│    3 customers = 1 deployment = 1x cost = 1x maintenance        │
│    But: you MUST enforce data isolation yourself.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 The Apartment Building Analogy

**This analogy will carry us through the strategies discussion.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE APARTMENT BUILDING                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Think of your SaaS application as REAL ESTATE.                 │
│                                                                 │
│  SINGLE-TENANT = Separate houses                                │
│  ──────────────                                                 │
│  Each customer gets their own building.                         │
│  Maximum privacy. Maximum cost.                                 │
│                                                                 │
│    🏠 Acme     🏠 Wayne     🏠 Stark                            │
│    Own land    Own land     Own land                             │
│    Own walls   Own walls    Own walls                            │
│    Own pipes   Own pipes    Own pipes                            │
│                                                                 │
│                                                                 │
│  MULTI-TENANT = Apartment building                              │
│  ─────────────                                                  │
│  Everyone shares the building. Each unit is private.            │
│  Shared infrastructure. Shared cost. Walls between units.       │
│                                                                 │
│    ┌─────────────────────────────┐                              │
│    │      Apartment Building     │                              │
│    │  ┌───────┬───────┬───────┐  │                              │
│    │  │ Acme  │ Wayne │ Stark │  │                              │
│    │  │ #301  │ #302  │ #303  │  │                              │
│    │  ├───────┴───────┴───────┤  │                              │
│    │  │   Shared lobby,       │  │                              │
│    │  │   plumbing, electric  │  │                              │
│    │  └───────────────────────┘  │                              │
│    └─────────────────────────────┘                              │
│                                                                 │
│  The BUILDING is your application.                              │
│  The APARTMENTS are tenant data spaces.                         │
│  The WALLS are your isolation enforcement.                      │
│  The LOBBY is shared infrastructure.                            │
│  The LOCKS on each door are your access controls.               │
│                                                                 │
│  Your job: make sure Apartment 301 can NEVER see               │
│  inside Apartment 302. Even if they share plumbing.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 The Core Challenge: Every Query, Every Time

**The single most important slide in this lecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│              THE MULTI-TENANCY MANTRA                            │
│                                                                 │
│     ╔═══════════════════════════════════════════════════╗        │
│     ║                                                   ║        │
│     ║          EVERY QUERY.  EVERY TIME.                ║        │
│     ║                                                   ║        │
│     ╚═══════════════════════════════════════════════════╝        │
│                                                                 │
│  Every SELECT must filter by tenant.                            │
│  Every INSERT must attach a tenant.                             │
│  Every UPDATE must verify the tenant.                           │
│  Every DELETE must scope to the tenant.                         │
│                                                                 │
│  Every endpoint. Every background job. Every migration.         │
│  Every report. Every export. Every search.                      │
│                                                                 │
│  Miss ONE query → data leak → lawsuit → game over.              │
│                                                                 │
│  This is not a feature you build once. It's a discipline        │
│  you enforce everywhere, forever.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Multi-tenancy is not a feature. It's an architectural property. You can't 'add multi-tenancy' to an endpoint. Either your entire system enforces tenant isolation, or it doesn't. There is no partial credit."

**Map this to what you've built:**

```
┌─────────────────────────────────────────────────────────────────┐
│           WHAT NEEDS TO BECOME TENANT-AWARE?                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Layer              │  Change Needed                            │
│  ───────────────────│───────────────────────────────────────    │
│  Database models    │  Add org_id column to every tenant table  │
│  (Week 6)           │                                           │
│                     │                                           │
│  Repository queries │  WHERE org_id = :current_org on EVERY     │
│  (Week 6)           │  query — no exceptions                    │
│                     │                                           │
│  API endpoints      │  Resolve tenant from request before       │
│  (Week 3-4)         │  any data access                          │
│                     │                                           │
│  Dependencies       │  New dependency: get_current_org()        │
│  (Week 3)           │  validates membership, returns org_id     │
│                     │                                           │
│  Background jobs    │  Explicitly pass org_id to every          │
│  (Week 11)          │  Celery task — no request context!        │
│                     │                                           │
│  Cache keys         │  Prefix EVERY cache key with org_id       │
│  (Week 10)          │  org:{id}:projects not just projects      │
│                     │                                           │
│  WebSocket rooms    │  Namespace channels by org_id             │
│  (Week 12)          │  org:{id}:notifications                   │
│                     │                                           │
└─────────────────────────────────────────────────────────────────┘
```

> "Look at that table. Everything you've built over 12 weeks gets touched. That's why multi-tenancy is an architecture lecture, not a feature lecture."

---

# PART 2: MULTI-TENANCY STRATEGIES

## 2.1 The Isolation Spectrum

**Three strategies, arranged from most isolated to least:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE ISOLATION SPECTRUM                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MORE ISOLATION                              LESS ISOLATION     │
│  MORE COST                                   LESS COST          │
│  ◀──────────────────────────────────────────────────────▶       │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐      │
│  │  DATABASE     │  │  SCHEMA      │  │  ROW-LEVEL        │      │
│  │  PER TENANT   │  │  PER TENANT  │  │  ISOLATION        │      │
│  │              │  │              │  │                   │      │
│  │  Separate    │  │  Shared DB,  │  │  Shared DB,       │      │
│  │  databases   │  │  separate    │  │  shared tables,   │      │
│  │  entirely    │  │  schemas     │  │  org_id column    │      │
│  └──────────────┘  └──────────────┘  └───────────────────┘      │
│                                                                 │
│  Separate houses    Separate floors    Shared floor plan         │
│  in a city          in a building      with assigned desks       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 Database-per-Tenant

```
┌─────────────────────────────────────────────────────────────────┐
│               DATABASE-PER-TENANT                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  acme_db     │  │  wayne_db    │  │  stark_db    │           │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │           │
│  │  │projects│  │  │  │projects│  │  │  │projects│  │           │
│  │  │tasks   │  │  │  │tasks   │  │  │  │tasks   │  │           │
│  │  │users   │  │  │  │users   │  │  │  │users   │  │           │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│       Host 1            Host 2            Host 3                │
│                                                                 │
│  ✅ ADVANTAGES:                                                  │
│  ├─ Maximum isolation (physically separate data)                │
│  ├─ Easy per-tenant backup/restore                              │
│  ├─ Per-tenant performance tuning                               │
│  ├─ Simple queries (no WHERE org_id clause needed)              │
│  └─ Compliance-friendly (data residency per country)            │
│                                                                 │
│  ❌ DISADVANTAGES:                                               │
│  ├─ Cost: 100 tenants = 100 database instances                  │
│  ├─ Migrations: must run Alembic on EVERY database              │
│  ├─ Cross-tenant queries impossible (analytics, admin)          │
│  ├─ Connection pooling nightmare (100 pools)                    │
│  └─ Onboarding: provisioning new DB per signup                  │
│                                                                 │
│  WHEN TO USE:                                                   │
│  ├─ Enterprise customers with strict compliance (healthcare)    │
│  ├─ Tenants with wildly different data sizes                    │
│  └─ Very few tenants (< 20), high revenue per tenant            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In code, this means swapping the database connection per request:**

```python
# Conceptual — you route to a different database per tenant
# Your app needs a registry of database URLs

TENANT_DATABASES = {
    "acme":  "postgresql+asyncpg://user:pass@host/acme_db",
    "wayne": "postgresql+asyncpg://user:pass@host/wayne_db",
}

# Connection routing happens BEFORE any query — typically in middleware
# This is complex to manage. We won't implement this approach.
```

---

## 2.3 Schema-per-Tenant

```
┌─────────────────────────────────────────────────────────────────┐
│               SCHEMA-PER-TENANT                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────────────────────────────────────────┐              │
│  │              ONE DATABASE                      │              │
│  │                                                │              │
│  │  ┌──────────────┐  ┌──────────────┐            │              │
│  │  │ schema:acme  │  │ schema:wayne │            │              │
│  │  │ ┌──────────┐ │  │ ┌──────────┐ │   ...      │              │
│  │  │ │ projects │ │  │ │ projects │ │            │              │
│  │  │ │ tasks    │ │  │ │ tasks    │ │            │              │
│  │  │ └──────────┘ │  │ └──────────┘ │            │              │
│  │  └──────────────┘  └──────────────┘            │              │
│  │                                                │              │
│  │  ┌──────────────┐                              │              │
│  │  │schema:public │  ← Shared tables             │              │
│  │  │ ┌──────────┐ │    (users, orgs, plans)      │              │
│  │  │ │ users    │ │                              │              │
│  │  │ │ orgs     │ │                              │              │
│  │  │ └──────────┘ │                              │              │
│  │  └──────────────┘                              │              │
│  └────────────────────────────────────────────────┘              │
│                                                                 │
│  ✅ ADVANTAGES:                                                  │
│  ├─ Good isolation (PostgreSQL schema = namespace)              │
│  ├─ Single database to manage                                   │
│  ├─ Shared tables possible (public schema)                      │
│  └─ Per-tenant backup possible (pg_dump --schema)               │
│                                                                 │
│  ❌ DISADVANTAGES:                                               │
│  ├─ Migrations on 500 schemas = slow and risky                  │
│  ├─ Schema count limits (PostgreSQL handles thousands,          │
│  │   but tooling gets slow)                                     │
│  ├─ SET search_path per request (easy to forget)                │
│  └─ ORMs don't always handle schema switching cleanly           │
│                                                                 │
│  WHEN TO USE:                                                   │
│  ├─ Medium tenant count (10-500)                                │
│  ├─ Need better isolation than row-level                        │
│  └─ Tenants have similar data shapes but strict boundaries      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# Conceptual — switch schema via SET search_path
# PostgreSQL allows schema switching per session

# In raw SQL:
# SET search_path TO 'acme', 'public';
# Now: SELECT * FROM projects → searches acme.projects first

# This is manageable but tricky with async connection pools.
# We won't implement this approach either.
```

---

## 2.4 Row-Level Isolation (Shared Everything)

**This is what we're building. It's the most common SaaS pattern.**

```
┌─────────────────────────────────────────────────────────────────┐
│               ROW-LEVEL ISOLATION                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────────────────────────────────────────┐              │
│  │           ONE DATABASE, ONE SCHEMA              │              │
│  │                                                │              │
│  │  projects TABLE:                               │              │
│  │  ┌────────┬──────────────────┬──────────────┐  │              │
│  │  │ org_id │ name             │ ...          │  │              │
│  │  ├────────┼──────────────────┼──────────────┤  │              │
│  │  │ acme   │ Website Redesign │ ...          │  │              │
│  │  │ acme   │ Q4 Earnings      │ ...          │  │              │
│  │  │ wayne  │ Project Gotham   │ ...          │  │              │
│  │  │ wayne  │ Arkham Upgrade   │ ...          │  │              │
│  │  │ stark  │ Arc Reactor v2   │ ...          │  │              │
│  │  └────────┴──────────────────┴──────────────┘  │              │
│  │       ▲                                        │              │
│  │       │                                        │              │
│  │  This column is the WALL between apartments.   │              │
│  │  Every query says: WHERE org_id = :current_org │              │
│  │                                                │              │
│  └────────────────────────────────────────────────┘              │
│                                                                 │
│  ✅ ADVANTAGES:                                                  │
│  ├─ Cheapest (one database, one set of tables)                  │
│  ├─ Simplest migrations (one Alembic history)                   │
│  ├─ Cross-tenant analytics easy (admin can query all)           │
│  ├─ Onboarding is instant (INSERT a row, not provision a DB)    │
│  ├─ Connection pooling works normally                           │
│  └─ Scales to thousands of tenants                              │
│                                                                 │
│  ❌ DISADVANTAGES:                                               │
│  ├─ Isolation is YOUR RESPONSIBILITY (the app must enforce it)  │
│  ├─ One bad query = cross-tenant data leak                      │
│  ├─ Noisy neighbor: one tenant's huge query slows everyone      │
│  ├─ Per-tenant backup requires filtered export                  │
│  └─ Compliance: all data in one DB (may not satisfy auditors)   │
│                                                                 │
│  WHEN TO USE:                                                   │
│  ├─ Most B2B SaaS applications (this is the default choice)     │
│  ├─ High tenant count (100s-1000s)                              │
│  ├─ Standard isolation requirements (not healthcare/gov)        │
│  └─ When simplicity and cost matter most                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Row-level isolation is the apartment building with locks on each door. The walls are made of WHERE clauses. If you forget to lock one door, anyone can walk in. That's the tradeoff — you get simplicity and cost efficiency, but the burden of enforcement is entirely on your code."

---

## 2.5 The Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│               WHICH STRATEGY DO I USE?                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                        START HERE                               │
│                            │                                    │
│                            ▼                                    │
│               ┌────────────────────────┐                        │
│               │ Do tenants need their  │                        │
│               │ own physical database? │                        │
│               │ (compliance, law,      │                        │
│               │  data residency)       │                        │
│               └────────────┬───────────┘                        │
│                  │                  │                            │
│                 YES                 NO                           │
│                  │                  │                            │
│                  ▼                  ▼                            │
│       ┌──────────────┐   ┌──────────────────┐                   │
│       │ DATABASE     │   │ Will you have    │                   │
│       │ PER TENANT   │   │ more than ~200   │                   │
│       └──────────────┘   │ tenants?         │                   │
│                          └────────┬─────────┘                   │
│                            │            │                       │
│                           YES           NO                      │
│                            │            │                       │
│                            ▼            ▼                       │
│               ┌──────────────┐  ┌────────────────┐              │
│               │  ROW-LEVEL   │  │ Schema-per or  │              │
│               │  ISOLATION   │  │ Row-level      │              │
│               │  ✅ OUR PICK │  │ (either works) │              │
│               └──────────────┘  └────────────────┘              │
│                                                                 │
│                                                                 │
│  FOR YOUR CAPSTONE: Row-level isolation.                        │
│  ├─ It's the most common pattern in real SaaS                   │
│  ├─ It uses tools you already know (SQLAlchemy, Depends)        │
│  ├─ It teaches the hardest lesson: disciplined data access      │
│  └─ It's what you'll encounter at most startups                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: IMPLEMENTING ROW-LEVEL ISOLATION

## 3.1 The Foundation: Organizations & Memberships

**In Lecture 1, you designed your domain model. Now we add the tenant layer.**

The key insight: a User doesn't "belong to" one organization — they can be a **member** of multiple organizations with different roles. Think of GitHub: you have a personal account, but you're a member of 3 different organizations, with admin rights in one and read-only in another.

**The relationship structure:**

```
┌─────────────────────────────────────────────────────────────────┐
│              USER ↔ ORGANIZATION RELATIONSHIP                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│        User                Membership           Organization    │
│    ┌──────────┐       ┌────────────────┐       ┌────────────┐   │
│    │ id       │──┐    │ id             │    ┌──│ id         │   │
│    │ email    │  │    │ user_id (FK) ──│────┘  │ name       │   │
│    │ password │  └────│── org_id (FK)  │       │ slug       │   │
│    │ name     │       │ role           │       │ created_at │   │
│    └──────────┘       │ joined_at      │       └────────────┘   │
│                       └────────────────┘                        │
│                                                                 │
│  One User → many Memberships → many Organizations               │
│  One Organization → many Memberships → many Users               │
│                                                                 │
│  This is the many-to-many with extra data pattern               │
│  you learned in Week 5 (junction tables).                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The models (building on Week 6 SQLAlchemy patterns):**

```python
# models/organization.py
import enum
import uuid
from datetime import datetime

from sqlalchemy import ForeignKey, String, UniqueConstraint
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.models.base import Base, TimestampMixin  # Your existing base from Week 6


class Organization(Base, TimestampMixin):
    __tablename__ = "organizations"

    id: Mapped[uuid.UUID] = mapped_column(
        primary_key=True, default=uuid.uuid4
    )
    name: Mapped[str] = mapped_column(String(255))
    slug: Mapped[str] = mapped_column(
        String(100), unique=True, index=True
    )

    # Relationships
    memberships: Mapped[list["Membership"]] = relationship(
        back_populates="organization", cascade="all, delete-orphan"
    )
```

```python
# models/membership.py

class OrgRole(str, enum.Enum):
    ADMIN = "admin"      # Full control over org settings + data
    MEMBER = "member"    # Can create/edit data within the org
    VIEWER = "viewer"    # Read-only access to org data


class Membership(Base, TimestampMixin):
    __tablename__ = "memberships"

    id: Mapped[uuid.UUID] = mapped_column(
        primary_key=True, default=uuid.uuid4
    )
    user_id: Mapped[uuid.UUID] = mapped_column(
        ForeignKey("users.id"), index=True
    )
    org_id: Mapped[uuid.UUID] = mapped_column(
        ForeignKey("organizations.id"), index=True
    )
    role: Mapped[OrgRole] = mapped_column(default=OrgRole.MEMBER)

    # A user can only be a member of a given org ONCE
    __table_args__ = (
        UniqueConstraint("user_id", "org_id", name="uq_user_org"),
    )

    # Relationships
    user: Mapped["User"] = relationship(back_populates="memberships")
    organization: Mapped["Organization"] = relationship(
        back_populates="memberships"
    )
```

> "Notice: `OrgRole` here is DIFFERENT from your Week 9 global RBAC roles. A user might be an app-level 'regular user' but an org-level 'admin' within Acme Corp. These are two separate axes of authorization. Global role = what can they do in the system. Org role = what can they do within this specific organization."

```
┌─────────────────────────────────────────────────────────────────┐
│              TWO AXES OF AUTHORIZATION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AXIS 1: GLOBAL ROLE (Week 9 — system-wide)                     │
│  ├─ super_admin  → can see all orgs, system settings            │
│  └─ user         → can only access orgs they belong to          │
│                                                                 │
│  AXIS 2: ORG ROLE (this lecture — per-organization)             │
│  ├─ admin   → manage org settings, invite/remove members        │
│  ├─ member  → create/edit/delete projects, tasks                │
│  └─ viewer  → read-only access to org data                      │
│                                                                 │
│  Example:                                                       │
│  Alice: global=user, Acme=admin, Wayne=viewer                   │
│  ├─ Can manage Acme Corp settings? YES (org admin)              │
│  ├─ Can create projects in Wayne? NO (org viewer)               │
│  └─ Can see system admin panel? NO (global user)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 The Tenant Column (org_id Everywhere)

**Every table that holds tenant-specific data needs an `org_id` column.**

> "Think of `org_id` as the apartment number stamped on every piece of furniture. When you query, you're saying 'show me only the furniture in apartment 302.' Without that stamp, you can't tell whose couch is whose."

```
┌─────────────────────────────────────────────────────────────────┐
│              WHICH TABLES GET org_id?                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TENANT-SCOPED (need org_id):      GLOBAL (no org_id):          │
│  ─────────────────────────         ────────────────────         │
│  ├─ projects                       ├─ users                     │
│  ├─ tasks                          ├─ organizations             │
│  ├─ comments                       ├─ memberships *             │
│  ├─ labels                         ├─ subscription_plans        │
│  ├─ attachments                    └─ system_settings           │
│  ├─ audit_logs                                                  │
│  └─ notifications                                               │
│                                                                 │
│  * memberships are global because they LINK users to orgs.      │
│    They don't "belong" to one org — they define the             │
│    relationship between users and orgs.                         │
│                                                                 │
│  RULE: If the data is created BY a tenant, FOR that tenant,     │
│        and should be invisible to other tenants → add org_id.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How the schema looks with org_id:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  projects                          tasks                        │
│  ┌───────────────────┐             ┌────────────────────┐       │
│  │ id          (PK)  │─────────┐   │ id          (PK)   │       │
│  │ org_id      (FK) ─│──┐      │   │ org_id      (FK) ──│──┐   │
│  │ name              │  │      └──▶│ project_id  (FK)   │  │   │
│  │ description       │  │          │ title              │  │   │
│  │ created_at        │  │          │ status             │  │   │
│  └───────────────────┘  │          │ assignee_id        │  │   │
│                         │          │ created_at         │  │   │
│  organizations          │          └────────────────────┘  │   │
│  ┌───────────────────┐  │                                  │   │
│  │ id          (PK) ◀│──┴──────────────────────────────────┘   │
│  │ name              │                                         │
│  │ slug              │  Every tenant table points back          │
│  └───────────────────┘  to organizations.id                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Yes, this means `org_id` appears in BOTH `projects` AND `tasks`, even though tasks already belong to a project (which has an org_id). This is intentional redundancy. Why? Because when you query tasks directly — not through a project — you need to filter by org_id without joining to the projects table. The denormalization is the price of safe, fast isolation."

---

## 3.3 Tenant-Aware Model Mixin

**Don't copy-paste `org_id` into every model. Use a mixin.**

```python
# models/mixins.py
import uuid

from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, declared_attr


class TenantMixin:
    """
    Add to any model that must be scoped to an organization.
    
    Provides:
    - org_id column with FK to organizations
    - Indexed for fast filtering (EVERY query will use this)
    
    Usage:
        class Project(Base, TenantMixin, TimestampMixin):
            ...
    """

    @declared_attr
    def org_id(cls) -> Mapped[uuid.UUID]:
        return mapped_column(
            ForeignKey("organizations.id"),
            index=True,       # You WILL filter on this constantly
            nullable=False,   # A tenant resource MUST belong to a tenant
        )
```

**Why `@declared_attr`?**

> "Remember decorators from Week 1? `@declared_attr` tells SQLAlchemy: 'don't evaluate this column definition right now — wait until a concrete model class uses this mixin, then create the column on THAT table.' Without it, SQLAlchemy would try to create the ForeignKey on the mixin itself, which has no table. Same concept as `@property`, but for SQLAlchemy schema definitions."

**Using the mixin (connection to Week 6 model definitions):**

```python
# models/project.py
import uuid

from sqlalchemy import String, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.models.base import Base, TimestampMixin
from app.models.mixins import TenantMixin


class Project(Base, TenantMixin, TimestampMixin):
    __tablename__ = "projects"

    id: Mapped[uuid.UUID] = mapped_column(
        primary_key=True, default=uuid.uuid4
    )
    name: Mapped[str] = mapped_column(String(255))
    description: Mapped[str | None] = mapped_column(Text, nullable=True)

    # org_id comes from TenantMixin — you don't see it here,
    # but it EXISTS on this table. Check with: Project.__table__.columns

    tasks: Mapped[list["Task"]] = relationship(
        back_populates="project", cascade="all, delete-orphan"
    )
```

```python
# models/task.py

class Task(Base, TenantMixin, TimestampMixin):
    __tablename__ = "tasks"

    id: Mapped[uuid.UUID] = mapped_column(
        primary_key=True, default=uuid.uuid4
    )
    project_id: Mapped[uuid.UUID] = mapped_column(
        ForeignKey("projects.id"), index=True
    )
    title: Mapped[str] = mapped_column(String(255))
    status: Mapped[str] = mapped_column(
        String(20), default="todo"
    )
    assignee_id: Mapped[uuid.UUID | None] = mapped_column(
        ForeignKey("users.id"), nullable=True
    )

    # org_id from TenantMixin — yes, ALSO here, not just on Project
    project: Mapped["Project"] = relationship(back_populates="tasks")
```

**The Alembic migration (connection to Week 6):**

```python
# After adding TenantMixin, run:
# alembic revision --autogenerate -m "add org_id to tenant tables"

# Alembic will generate:
def upgrade():
    op.add_column('projects', sa.Column(
        'org_id', sa.UUID(), nullable=False
    ))
    op.create_index(
        op.f('ix_projects_org_id'), 'projects', ['org_id']
    )
    op.create_foreign_key(
        'fk_projects_org_id', 'projects',
        'organizations', ['org_id'], ['id']
    )
    # Same for tasks, comments, labels...
```

> "If you already have data in these tables, this migration will fail — you can't add a NOT NULL column without a default. In production, you'd do a multi-step migration: add the column as nullable, backfill the data, then alter to NOT NULL. You practiced migration patterns in Week 6."

---

## 3.4 Resolving Tenant Context (The Dependency Chain)

**The critical question: when a request comes in, how do you know which organization the user is acting within?**

```
┌─────────────────────────────────────────────────────────────────┐
│            TENANT CONTEXT RESOLUTION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP Request arrives:                                          │
│  ┌──────────────────────────────────────────────────┐           │
│  │ GET /api/v1/projects                              │           │
│  │ Authorization: Bearer eyJhbGciOi...               │           │
│  │ X-Org-ID: 550e8400-e29b-41d4-a716-446655440000   │           │
│  └──────────────────────────────────────────────────┘           │
│                                                                 │
│  The dependency chain:                                          │
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐      │
│  │ JWT Token   │───▶│ get_current │───▶│ get_current_org │      │
│  │ (header)    │    │ _user()     │    │ ()              │      │
│  └─────────────┘    │ (Week 9)    │    │ (NEW)           │      │
│                     └─────────────┘    └────────┬────────┘      │
│                                                 │               │
│       Step 1: WHO are you?          Step 2: WHICH org?          │
│       (Authentication)              (Tenant Resolution)         │
│                                                 │               │
│                                                 ▼               │
│                                       ┌─────────────────┐       │
│                                       │ Validate: is    │       │
│                                       │ this user a     │       │
│                                       │ member of this  │       │
│                                       │ org?            │       │
│                                       └────────┬────────┘       │
│                                           │         │           │
│                                          YES        NO          │
│                                           │         │           │
│                                           ▼         ▼           │
│                                     Return     403              │
│                                     org_id     Forbidden        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why a header (`X-Org-ID`) and not something else?**

```
┌─────────────────────────────────────────────────────────────────┐
│          APPROACHES TO PASSING TENANT CONTEXT                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPTION A: Header (X-Org-ID)          ← OUR CHOICE              │
│  ├─ Clean URLs: GET /projects                                   │
│  ├─ Easy to switch orgs (change header)                         │
│  ├─ Used by: Stripe (Stripe-Account), Shopify                  │
│  └─ Works for ALL endpoints uniformly                           │
│                                                                 │
│  OPTION B: URL path (/orgs/{org_id}/projects)                   │
│  ├─ RESTful: resource nesting                                   │
│  ├─ Visible in URL: clear which org                             │
│  ├─ Verbose: every route needs org_id prefix                    │
│  └─ Used by: GitHub API (/orgs/{org}/repos)                     │
│                                                                 │
│  OPTION C: JWT claim (org_id baked into the token)              │
│  ├─ No extra header/parameter                                   │
│  ├─ Inflexible: user must re-login to switch orgs               │
│  └─ ❌ Breaks if user is a member of multiple orgs              │
│                                                                 │
│  OPTION D: Subdomain (acme.yourapp.com)                         │
│  ├─ Clean separation, familiar UX                               │
│  ├─ Complex: DNS, TLS certs per tenant                          │
│  └─ Used by: Slack (acme.slack.com), Jira                       │
│                                                                 │
│  We use the header approach. It's clean, flexible,              │
│  and straightforward to implement with Depends().               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The dependency implementation:**

```python
# dependencies/tenant.py
import uuid

from fastapi import Depends, HTTPException, Request
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies.auth import get_current_user  # Your Week 9 dependency
from app.dependencies.database import get_session    # Your Week 6 dependency
from app.models.membership import Membership
from app.models.user import User


async def get_current_org(
    request: Request,
    current_user: User = Depends(get_current_user),
    session: AsyncSession = Depends(get_session),
) -> uuid.UUID:
    """
    Extract tenant context from request and validate membership.
    
    This dependency answers: "Which organization is this user
    acting on behalf of RIGHT NOW?"
    
    Returns org_id (not the full Organization object) because
    that's what repositories need for filtering.
    """

    # Step 1: Extract org_id from header
    org_id_raw = request.headers.get("X-Org-ID")

    if not org_id_raw:
        raise HTTPException(
            status_code=400,
            detail="X-Org-ID header is required. "
                   "Specify which organization you are acting within.",
        )

    # Step 2: Validate format
    try:
        org_id = uuid.UUID(org_id_raw)
    except ValueError:
        raise HTTPException(
            status_code=400,
            detail="X-Org-ID must be a valid UUID.",
        )

    # Step 3: Verify the user is a member of this organization
    stmt = select(Membership).where(
        Membership.user_id == current_user.id,
        Membership.org_id == org_id,
    )
    result = await session.execute(stmt)
    membership = result.scalar_one_or_none()

    if membership is None:
        # Don't reveal whether the org EXISTS or not.
        # Just say "forbidden." (Security: info leakage prevention)
        raise HTTPException(
            status_code=403,
            detail="You do not have access to this organization.",
        )

    return org_id
```

**Now ask the class:**

> "Look at Step 3 carefully. Why do we check membership in the DATABASE every time, instead of storing org memberships in the JWT? Think about it for a moment."

Answer: **Because memberships can change.** If an admin removes Alice from Acme Corp at 2:00 PM, but her JWT doesn't expire until 3:00 PM, she still has a valid token claiming she's in Acme. By checking the database, we ensure the membership is current *at the time of the request*. Security over convenience.

**Getting the org role too (for org-level RBAC):**

```python
# When you need the role as well — for permission checks within the org

from app.models.membership import Membership, OrgRole

async def get_current_membership(
    request: Request,
    current_user: User = Depends(get_current_user),
    session: AsyncSession = Depends(get_session),
) -> Membership:
    """
    Like get_current_org, but returns the full Membership object.
    Use this when you need the user's ROLE within the org.
    """
    org_id_raw = request.headers.get("X-Org-ID")

    if not org_id_raw:
        raise HTTPException(status_code=400, detail="X-Org-ID header required.")

    try:
        org_id = uuid.UUID(org_id_raw)
    except ValueError:
        raise HTTPException(status_code=400, detail="Invalid X-Org-ID format.")

    stmt = select(Membership).where(
        Membership.user_id == current_user.id,
        Membership.org_id == org_id,
    )
    result = await session.execute(stmt)
    membership = result.scalar_one_or_none()

    if membership is None:
        raise HTTPException(status_code=403, detail="No access to this organization.")

    return membership


# Use it for role-gated endpoints:

def require_org_role(minimum_role: OrgRole):
    """
    Dependency factory: require at least this role in the current org.
    
    Role hierarchy: ADMIN > MEMBER > VIEWER
    """
    role_hierarchy = {
        OrgRole.VIEWER: 0,
        OrgRole.MEMBER: 1,
        OrgRole.ADMIN: 2,
    }

    async def check_role(
        membership: Membership = Depends(get_current_membership),
    ) -> Membership:
        if role_hierarchy[membership.role] < role_hierarchy[minimum_role]:
            raise HTTPException(
                status_code=403,
                detail=f"Requires at least '{minimum_role.value}' role in this organization.",
            )
        return membership

    return check_role
```

**Usage in a route:**

```python
@router.delete("/projects/{project_id}")
async def delete_project(
    project_id: uuid.UUID,
    membership: Membership = Depends(require_org_role(OrgRole.ADMIN)),
    repo: ProjectRepository = Depends(get_project_repo),
):
    """Only org admins can delete projects."""
    project = await repo.get_by_id(project_id)
    if not project:
        raise HTTPException(status_code=404, detail="Project not found.")
    await repo.soft_delete(project_id)
    return {"detail": "Project deleted."}
```

**The full dependency chain visualized:**

```
┌─────────────────────────────────────────────────────────────────┐
│            THE COMPLETE DEPENDENCY CHAIN                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP Request                                                   │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────┐                                           │
│  │  get_session()   │  ← Week 6: DB session per request         │
│  └────────┬─────────┘                                           │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────┐                                           │
│  │ get_current_user │  ← Week 9: JWT → User object              │
│  └────────┬─────────┘                                           │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────┐                                           │
│  │ get_current_org  │  ← NEW: header → validate → org_id        │
│  └────────┬─────────┘                                           │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────┐                                           │
│  │ get_project_repo │  ← Week 6: repository scoped to org_id    │
│  └────────┬─────────┘                                           │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────┐                                           │
│  │  Route Handler   │  ← Receives a repo that can ONLY see      │
│  │  (your endpoint) │    the current tenant's data               │
│  └──────────────────┘                                           │
│                                                                 │
│  By the time the route handler runs, isolation is ALREADY       │
│  enforced. The handler can't accidentally query all tenants     │
│  because the repo won't let it.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Tenant-Scoped Repositories

**The repository pattern (Week 6) becomes tenant-aware. This is where the real enforcement happens.**

> "Remember the repository pattern? It's a class that encapsulates all database queries for a model. Now we make ONE change: every method's base query includes `WHERE org_id = :current_org`. This single change protects your entire data layer."

```python
# repositories/base.py
import uuid
from typing import Generic, TypeVar, Sequence

from sqlalchemy import select, Select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.base import Base
from app.models.mixins import TenantMixin

T = TypeVar("T", bound=Base)


class TenantRepository(Generic[T]):
    """
    Base repository for tenant-scoped models.
    
    EVERY query method calls _base_query(), which ALWAYS
    filters by org_id. There is no way to accidentally
    query across tenants through this repository.
    """

    def __init__(
        self,
        model: type[T],
        session: AsyncSession,
        org_id: uuid.UUID,
    ):
        self.model = model
        self.session = session
        self.org_id = org_id

    def _base_query(self) -> Select:
        """
        THE CRITICAL METHOD.
        
        Every query starts here. Every query is scoped.
        This is the lock on the apartment door.
        """
        return select(self.model).where(
            self.model.org_id == self.org_id
        )

    async def get_by_id(self, entity_id: uuid.UUID) -> T | None:
        stmt = self._base_query().where(self.model.id == entity_id)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def get_all(
        self,
        offset: int = 0,
        limit: int = 50,
    ) -> Sequence[T]:
        stmt = self._base_query().offset(offset).limit(limit)
        result = await self.session.execute(stmt)
        return result.scalars().all()

    async def create(self, **kwargs) -> T:
        # org_id is INJECTED by the repository, not the caller.
        # The route handler never touches org_id directly.
        entity = self.model(org_id=self.org_id, **kwargs)
        self.session.add(entity)
        await self.session.flush()
        return entity

    async def update(
        self,
        entity_id: uuid.UUID,
        **kwargs,
    ) -> T | None:
        entity = await self.get_by_id(entity_id)  # ← scoped query!
        if entity is None:
            return None
        for key, value in kwargs.items():
            setattr(entity, key, value)
        await self.session.flush()
        return entity

    async def delete(self, entity_id: uuid.UUID) -> bool:
        entity = await self.get_by_id(entity_id)  # ← scoped query!
        if entity is None:
            return False
        await self.session.delete(entity)
        await self.session.flush()
        return True
```

**Notice three things:**

```
┌─────────────────────────────────────────────────────────────────┐
│            THREE GUARANTEES OF THIS REPOSITORY                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. READS ARE SCOPED                                            │
│     _base_query() adds WHERE org_id = :current_org              │
│     get_by_id() uses _base_query() → can't read other           │
│     tenant's data even if you know the UUID                     │
│                                                                 │
│  2. WRITES ARE SCOPED                                           │
│     create() injects org_id automatically                       │
│     The caller NEVER passes org_id — they can't set it wrong    │
│                                                                 │
│  3. DELETES ARE SCOPED                                          │
│     delete() calls get_by_id() first → if the entity doesn't   │
│     belong to this org, get_by_id returns None → delete is      │
│     silently blocked. No cross-tenant deletion possible.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Specialized repositories extend the base:**

```python
# repositories/project.py

class ProjectRepository(TenantRepository[Project]):
    """Project-specific queries, all tenant-scoped."""

    def __init__(self, session: AsyncSession, org_id: uuid.UUID):
        super().__init__(Project, session, org_id)

    async def get_by_slug(self, slug: str) -> Project | None:
        # _base_query() ensures org_id scoping even here
        stmt = self._base_query().where(Project.slug == slug)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def get_with_tasks(self, project_id: uuid.UUID) -> Project | None:
        # Eager-load tasks (Week 6 — joinedload to avoid N+1)
        stmt = (
            self._base_query()
            .where(Project.id == project_id)
            .options(selectinload(Project.tasks))
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def search(self, query: str) -> Sequence[Project]:
        stmt = self._base_query().where(
            Project.name.ilike(f"%{query}%")
        )
        result = await self.session.execute(stmt)
        return result.scalars().all()
```

**Wiring it up as a FastAPI dependency:**

```python
# dependencies/repositories.py

def get_project_repo(
    session: AsyncSession = Depends(get_session),
    org_id: uuid.UUID = Depends(get_current_org),
) -> ProjectRepository:
    """
    Factory dependency: creates a ProjectRepository
    already scoped to the current tenant.
    
    The route handler receives a repo that can ONLY
    access the current org's projects. No way to cheat.
    """
    return ProjectRepository(session=session, org_id=org_id)


def get_task_repo(
    session: AsyncSession = Depends(get_session),
    org_id: uuid.UUID = Depends(get_current_org),
) -> TaskRepository:
    return TaskRepository(session=session, org_id=org_id)
```

**The route handler becomes clean and safe:**

```python
# routes/projects.py

@router.get("/projects", response_model=list[ProjectResponse])
async def list_projects(
    repo: ProjectRepository = Depends(get_project_repo),
):
    # This handler has NO IDEA what org_id is.
    # It can't query the wrong tenant even if it tries.
    # The repo handles everything.
    return await repo.get_all()


@router.post("/projects", response_model=ProjectResponse, status_code=201)
async def create_project(
    data: ProjectCreate,
    repo: ProjectRepository = Depends(get_project_repo),
):
    # org_id is injected by the repo's create() method.
    # The caller only passes the project data.
    project = await repo.create(
        name=data.name,
        description=data.description,
    )
    return project
```

---

## 3.6 The Horror: One Missing WHERE Clause

**Let's see what happens when someone bypasses the pattern.**

```python
# ❌ A developer bypasses the repository "just this once"
# "I just need a quick query, the repo doesn't have this method yet..."

@router.get("/reports/all-tasks")
async def task_report(
    session: AsyncSession = Depends(get_session),
    current_user: User = Depends(get_current_user),
):
    # "I'll just grab all tasks for the current user"
    stmt = select(Task).where(Task.assignee_id == current_user.id)
    result = await session.execute(stmt)
    return result.scalars().all()
    
    # PROBLEM: No org_id filter.
    # If Alice is a member of Acme AND Wayne (two memberships),
    # this returns her tasks from BOTH orgs.
    # The response is sent in the context of whatever org the
    # X-Org-ID header says — but the DATA crosses boundaries.
```

**The consequence:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE DATA LEAK                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Alice is a member of Acme Corp AND Wayne Enterprises.          │
│  She's logged into the Acme dashboard (X-Org-ID: acme).        │
│                                                                 │
│  GET /reports/all-tasks                                         │
│                                                                 │
│  Response (WRONG — crosses tenant boundary):                    │
│  [                                                              │
│    {"title": "Update website",      "org": "Acme Corp"},       │
│    {"title": "File Q4 report",      "org": "Acme Corp"},       │
│    {"title": "Review Gotham specs",  "org": "Wayne Ent."},     │
│                                      ▲                         │
│                                      │                         │
│                           Wayne data visible to Acme dashboard! │
│  ]                                                              │
│                                                                 │
│  Not only can Alice see her Wayne tasks from within Acme's      │
│  dashboard — if she shares her screen, her Acme Corp            │
│  colleagues now see Wayne Enterprises' project names.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to prevent this from ever happening:**

```
┌─────────────────────────────────────────────────────────────────┐
│              PREVENTION STRATEGIES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ALWAYS USE THE REPOSITORY                                   │
│     Never write raw select(Model) in route handlers.            │
│     The repository is the ONLY door to the database.            │
│     If you need a new query, add a method to the repository.    │
│                                                                 │
│  2. CODE REVIEW CHECKLIST                                       │
│     Every PR: "Does any endpoint query the DB without going     │
│     through a tenant-scoped repository?"                        │
│     If yes → reject the PR.                                     │
│                                                                 │
│  3. WRITE ISOLATION TESTS                                       │
│     For every endpoint, test with two orgs:                     │
│     ├─ Create data in Org A                                     │
│     ├─ Query from Org B                                         │
│     └─ Assert: Org B sees NOTHING from Org A                    │
│                                                                 │
│  4. ADVANCED: PostgreSQL Row-Level Security (RLS)               │
│     Database-level enforcement. Even raw SQL respects it.       │
│     Out of scope for us, but know it exists for production.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The isolation test pattern (connection to Week 4 — testing APIs):**

```python
# tests/test_tenant_isolation.py

import uuid
import pytest

# Suppose you have test fixtures that create two orgs
# and users who belong to each (from your conftest.py)


async def test_org_a_cannot_see_org_b_projects(
    async_client,                   # httpx.AsyncClient from Week 4
    org_a_id: uuid.UUID,
    org_b_id: uuid.UUID,
    org_a_user_token: str,
    org_b_user_token: str,
):
    """THE most important test in a multi-tenant app."""

    # Step 1: Create a project in Org B
    response = await async_client.post(
        "/api/v1/projects",
        json={"name": "Wayne Secret Project"},
        headers={
            "Authorization": f"Bearer {org_b_user_token}",
            "X-Org-ID": str(org_b_id),
        },
    )
    assert response.status_code == 201

    # Step 2: List projects from Org A
    response = await async_client.get(
        "/api/v1/projects",
        headers={
            "Authorization": f"Bearer {org_a_user_token}",
            "X-Org-ID": str(org_a_id),
        },
    )
    assert response.status_code == 200

    projects = response.json()

    # Step 3: Org A must see ZERO projects (they didn't create any)
    assert len(projects) == 0  # ← If this fails, you have a data leak.

    # Bonus: verify by name as a safeguard
    project_names = [p["name"] for p in projects]
    assert "Wayne Secret Project" not in project_names
```

> "Write this test for EVERY tenant-scoped endpoint. It's repetitive. It's boring. It will save your company."

---

# PART 4: SHARED VS TENANT-SPECIFIC RESOURCES

## 4.1 Drawing the Line

Not everything in your application belongs to a tenant. Some resources are global — they exist outside the tenant boundary and are shared across the entire system.

> "Back to the apartment building: the mailboxes are per-apartment (tenant-scoped). The elevator is shared (global). The building rules posted in the lobby are shared. But the furniture inside each unit is private."

```
┌─────────────────────────────────────────────────────────────────┐
│             THE CLASSIFICATION RULE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Ask two questions:                                             │
│                                                                 │
│  1. "Was this created BY a specific organization?"              │
│  2. "Should it be invisible to other organizations?"            │
│                                                                 │
│  Both YES → Tenant-scoped (needs org_id)                        │
│  Either NO → Global (no org_id)                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Classification Examples

```
┌─────────────────────────────────────────────────────────────────┐
│           RESOURCE CLASSIFICATION                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GLOBAL (no org_id):                                            │
│  ─────────────────                                              │
│  ├─ Users              Created at registration, exist globally  │
│  │                     A user can belong to many orgs.          │
│  │                                                              │
│  ├─ Organizations      The tenant itself — can't belong to      │
│  │                     another tenant.                          │
│  │                                                              │
│  ├─ Memberships        Link users to orgs. They ARE the         │
│  │                     boundary, not inside it.                 │
│  │                                                              │
│  ├─ Subscription Plans Defined by YOU (the SaaS provider).      │
│  │                     "Free", "Pro", "Enterprise" — same       │
│  │                     for all tenants.                         │
│  │                                                              │
│  └─ System Settings    App-wide feature flags, maintenance      │
│                        mode, global rate limits.                │
│                                                                 │
│                                                                 │
│  TENANT-SCOPED (needs org_id):                                  │
│  ─────────────────────────────                                  │
│  ├─ Projects           Created by an org, private to that org.  │
│  ├─ Tasks              Belong to a project, which belongs       │
│  │                     to an org.                               │
│  ├─ Comments           Written within an org's context.         │
│  ├─ Labels/Tags        Org-specific categorization.             │
│  ├─ Audit Logs         Record of changes within an org.         │
│  ├─ Notifications      Triggered by org activity.               │
│  ├─ Invitations        Org-specific invites to join.            │
│  └─ File Attachments   Uploaded by org members.                 │
│                                                                 │
│                                                                 │
│  GRAY AREA (design decision):                                   │
│  ──────────────────────────────                                 │
│  ├─ Templates          System-provided defaults? Global.        │
│  │                     Org-customized templates? Tenant-scoped. │
│  │                     Both? Two tables, or a "source" field.   │
│  │                                                              │
│  └─ Notification       Global defaults (system-wide),           │
│     Preferences        but overridable per org? Two layers.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Modeling the Boundary in Code

**Show the difference clearly in your model layer:**

```python
# GLOBAL models: no TenantMixin
class User(Base, TimestampMixin):
    __tablename__ = "users"
    # No org_id — users are global
    ...

class SubscriptionPlan(Base, TimestampMixin):
    __tablename__ = "subscription_plans"
    # No org_id — plans are system-wide
    ...


# TENANT models: use TenantMixin
class Project(Base, TenantMixin, TimestampMixin):
    __tablename__ = "projects"
    # org_id comes from TenantMixin
    ...

class Task(Base, TenantMixin, TimestampMixin):
    __tablename__ = "tasks"
    # org_id comes from TenantMixin
    ...


# GRAY AREA: explicit source tracking
class ProjectTemplate(Base, TimestampMixin):
    __tablename__ = "project_templates"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(255))
    
    # Nullable org_id: None = system-provided, set = org-custom
    org_id: Mapped[uuid.UUID | None] = mapped_column(
        ForeignKey("organizations.id"), nullable=True, index=True
    )
    is_system: Mapped[bool] = mapped_column(default=False)
```

```python
# Query logic for gray-area resources:
async def get_available_templates(
    session: AsyncSession,
    org_id: uuid.UUID,
) -> Sequence[ProjectTemplate]:
    """Return system templates + this org's custom templates."""
    stmt = select(ProjectTemplate).where(
        or_(
            ProjectTemplate.is_system == True,        # System templates
            ProjectTemplate.org_id == org_id,          # This org's customs
        )
    )
    result = await session.execute(stmt)
    return result.scalars().all()
```

> "The gray area is where you make design decisions. Document them in your ADRs (Lecture 1). 'ADR-004: Project templates are globally available, but custom templates are org-scoped.' Future developers will thank you."

---

# PART 5: AUDIT LOGGING

## 5.1 The "Who Changed This?" Problem

**Start with a scenario:**

> "It's 3 AM. Your on-call phone buzzes. A customer says: 'Someone deleted our entire Q4 project with 200 tasks. We don't know who did it or when. Can you recover it?'"

```
┌─────────────────────────────────────────────────────────────────┐
│                 WITHOUT AUDIT LOGGING:                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Customer: "Who deleted our project?"                           │
│  You: "I... don't know."                                        │
│                                                                 │
│  Customer: "When was it deleted?"                               │
│  You: "I don't know that either."                               │
│                                                                 │
│  Customer: "Can you restore it?"                                │
│  You: "It was a hard DELETE. It's gone."                         │
│                                                                 │
│  Customer: "We're switching to your competitor."                 │
│  You: "..."                                                     │
│                                                                 │
│                                                                 │
│                 WITH AUDIT LOGGING:                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Customer: "Who deleted our project?"                           │
│  You: "Bob Smith, at 2:47 AM from IP 203.0.113.42."            │
│                                                                 │
│  Customer: "Can you restore it?"                                │
│  You: "Already done. It was soft-deleted. Everything's back."   │
│                                                                 │
│  Customer: "How did you know all that?"                         │
│  You: "We log every change. Here's the full timeline."          │
│                                                                 │
│  Customer: "We love your product."                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Audit logs serve three purposes:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY AUDIT LOG?                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. DEBUGGING                                                   │
│     "What happened?" → The audit trail tells you exactly        │
│     what changed, when, and by whom.                            │
│                                                                 │
│  2. COMPLIANCE                                                  │
│     SOC 2, GDPR, HIPAA — all require an audit trail.            │
│     "Can you prove who accessed this data?"                     │
│     "Can you show all changes to customer records?"             │
│                                                                 │
│  3. ACCOUNTABILITY                                              │
│     Tenant admins want to see: "What did my team do today?"     │
│     Activity feeds, change histories, "last modified by"        │
│     are all features powered by audit logs.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Anatomy of an Audit Event

**Every audit entry answers the 5 W's:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE FIVE W's OF AN AUDIT EVENT                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHO?    user_id     → Which user performed the action          │
│  WHAT?   action      → "created", "updated", "deleted"         │
│  WHICH?  entity_type → "project", "task", "member"             │
│          entity_id   → UUID of the specific record              │
│  WHEN?   timestamp   → Exact time of the action (UTC)          │
│  WHERE?  org_id      → Which tenant's data was affected         │
│          ip_address  → Client IP (useful for security audits)   │
│                                                                 │
│  BONUS:                                                         │
│  HOW?    changes     → {"name": {"old": "Alpha", "new": "Beta"}}│
│          (what specifically changed — field-level diff)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**An example audit trail:**

```
┌──────────┬──────┬─────────┬────────────┬─────────────────────────┐
│ timestamp│ user │ action  │ entity     │ changes                 │
├──────────┼──────┼─────────┼────────────┼─────────────────────────┤
│ 14:01:03 │ Alice│ created │ project/7a │ {name: "Q4 Plan"}       │
│ 14:05:22 │ Alice│ updated │ project/7a │ {desc: {old:"", new:…}} │
│ 14:12:44 │ Bob  │ created │ task/3b    │ {title: "Draft report"} │
│ 14:30:01 │ Alice│ updated │ task/3b    │ {status: {old:"todo",   │
│          │      │         │            │  new:"in_progress"}}    │
│ 15:02:18 │ Bob  │ deleted │ task/3b    │ null (soft delete)      │
└──────────┴──────┴─────────┴────────────┴─────────────────────────┘
```

---

## 5.3 The Audit Log Table

```python
# models/audit.py
import uuid
from datetime import datetime

from sqlalchemy import ForeignKey, String, Text
from sqlalchemy.dialects.postgresql import JSONB  # Week 5 — PostgreSQL JSONB
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base


class AuditLog(Base):
    """
    Append-only audit trail. NEVER update or delete these records.
    
    This table is tenant-scoped (has org_id) so tenants
    can view their own activity history.
    """
    __tablename__ = "audit_logs"

    id: Mapped[uuid.UUID] = mapped_column(
        primary_key=True, default=uuid.uuid4
    )

    # TENANT SCOPE — which org was this action within?
    org_id: Mapped[uuid.UUID] = mapped_column(
        ForeignKey("organizations.id"), index=True
    )

    # WHO performed the action
    user_id: Mapped[uuid.UUID] = mapped_column(
        ForeignKey("users.id"), index=True
    )

    # WHAT action was performed
    action: Mapped[str] = mapped_column(String(20))
    # "created" | "updated" | "deleted" | "restored"

    # WHICH entity was affected
    entity_type: Mapped[str] = mapped_column(String(50), index=True)
    # "project" | "task" | "member" | "label"
    entity_id: Mapped[uuid.UUID] = mapped_column(index=True)

    # HOW it changed (JSONB — stores the diff)
    changes: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    # Example: {"name": {"old": "Alpha", "new": "Beta"}}

    # CONTEXT
    ip_address: Mapped[str | None] = mapped_column(
        String(45), nullable=True  # 45 chars covers IPv6
    )

    # WHEN (not using TimestampMixin — audit logs don't get "updated")
    created_at: Mapped[datetime] = mapped_column(
        default=datetime.utcnow, index=True
    )
```

**Why no `updated_at` on audit logs?**

> "Audit logs are APPEND-ONLY. You write them once, you never modify them. If you could edit an audit log, it would defeat the purpose — it's like letting suspects edit the security camera footage. No `updated_at`, no `UPDATE` queries, no `DELETE`. Ever."

**Important indexes (connection to Week 7 — query optimization):**

```python
    # In __table_args__, add a composite index for common queries:
    __table_args__ = (
        # "Show me all activity in this org, most recent first"
        # This is the most common audit query
        Index("ix_audit_org_created", "org_id", "created_at"),

        # "Show me everything that happened to this specific project"
        Index("ix_audit_entity", "entity_type", "entity_id"),
    )
```

---

## 5.4 Recording Changes (Service Layer)

**The audit service sits in your service layer, called explicitly:**

```python
# services/audit.py
import uuid
from datetime import datetime

from sqlalchemy.ext.asyncio import AsyncSession

from app.models.audit import AuditLog


class AuditService:
    """
    Records audit events for a specific tenant and user.
    
    Created per-request via dependency injection.
    Writes happen within the same database transaction
    as the action itself — if the action rolls back,
    the audit log rolls back too.
    """

    def __init__(
        self,
        session: AsyncSession,
        org_id: uuid.UUID,
        user_id: uuid.UUID,
        ip_address: str | None = None,
    ):
        self.session = session
        self.org_id = org_id
        self.user_id = user_id
        self.ip_address = ip_address

    async def log(
        self,
        action: str,
        entity_type: str,
        entity_id: uuid.UUID,
        changes: dict | None = None,
    ) -> AuditLog:
        entry = AuditLog(
            org_id=self.org_id,
            user_id=self.user_id,
            action=action,
            entity_type=entity_type,
            entity_id=entity_id,
            changes=changes,
            ip_address=self.ip_address,
        )
        self.session.add(entry)
        # Don't commit — let the request's transaction handle it.
        # If the main operation fails, the audit log fails too.
        # This is CORRECT — you don't want to log an action
        # that didn't actually happen.
        await self.session.flush()
        return entry
```

**The helper for computing field-level diffs:**

```python
# services/audit.py (continued)

def compute_changes(
    before: dict,
    after: dict,
    exclude_fields: set[str] | None = None,
) -> dict:
    """
    Compare two states and return the diff.
    
    Before: {"name": "Alpha", "status": "todo", "updated_at": "..."}
    After:  {"name": "Beta",  "status": "todo", "updated_at": "..."}
    
    Returns: {"name": {"old": "Alpha", "new": "Beta"}}
    (status unchanged → not included. updated_at excluded.)
    """
    exclude = exclude_fields or {"updated_at", "created_at"}
    changes = {}

    all_keys = set(before.keys()) | set(after.keys())
    for key in all_keys:
        if key in exclude:
            continue
        old_val = before.get(key)
        new_val = after.get(key)
        if old_val != new_val:
            changes[key] = {"old": old_val, "new": new_val}

    return changes
```

**Wiring the audit service as a dependency:**

```python
# dependencies/audit.py

async def get_audit_service(
    request: Request,
    session: AsyncSession = Depends(get_session),
    org_id: uuid.UUID = Depends(get_current_org),
    current_user: User = Depends(get_current_user),
) -> AuditService:
    return AuditService(
        session=session,
        org_id=org_id,
        user_id=current_user.id,
        ip_address=request.client.host if request.client else None,
    )
```

**Using it in a route handler:**

```python
@router.put("/projects/{project_id}", response_model=ProjectResponse)
async def update_project(
    project_id: uuid.UUID,
    data: ProjectUpdate,
    repo: ProjectRepository = Depends(get_project_repo),
    audit: AuditService = Depends(get_audit_service),
):
    # Step 1: Get current state (BEFORE the change)
    project = await repo.get_by_id(project_id)
    if not project:
        raise HTTPException(status_code=404, detail="Project not found.")

    before = {"name": project.name, "description": project.description}

    # Step 2: Apply the update
    updated = await repo.update(
        project_id,
        **data.model_dump(exclude_unset=True),
    )

    after = {"name": updated.name, "description": updated.description}

    # Step 3: Log the change
    changes = compute_changes(before, after)
    if changes:  # Only log if something actually changed
        await audit.log(
            action="updated",
            entity_type="project",
            entity_id=project_id,
            changes=changes,
        )

    return updated
```

**The flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  AUDIT LOGGING FLOW                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Route Handler                                                  │
│       │                                                         │
│       │ ① Get current state (before)                            │
│       ▼                                                         │
│  ┌──────────┐                                                   │
│  │   Repo   │──── SELECT ... WHERE org_id = ? AND id = ?        │
│  └──────────┘                                                   │
│       │                                                         │
│       │ ② Apply update                                          │
│       ▼                                                         │
│  ┌──────────┐                                                   │
│  │   Repo   │──── UPDATE ... SET name = ? WHERE org_id = ? ...  │
│  └──────────┘                                                   │
│       │                                                         │
│       │ ③ Compute diff + log                                    │
│       ▼                                                         │
│  ┌──────────┐                                                   │
│  │  Audit   │──── INSERT INTO audit_logs (...)                  │
│  │  Service │                                                   │
│  └──────────┘                                                   │
│       │                                                         │
│       │ ④ Transaction commits (all or nothing)                  │
│       ▼                                                         │
│  ┌──────────┐                                                   │
│  │  COMMIT  │──── Data change + audit log saved atomically      │
│  └──────────┘                                                   │
│                                                                 │
│  If the UPDATE fails → audit log is NOT saved (correct!)        │
│  If the audit INSERT fails → data change is rolled back too     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 Async Audit Logging (Background Processing)

**The approach above writes the audit log synchronously — in the same transaction. This is correct for critical audit trails. But for high-traffic systems, you might not want every request to do an extra INSERT.**

> "This is a tradeoff. Synchronous audit logging guarantees consistency — if the action happened, the log exists. Async audit logging improves latency but introduces a small risk of losing log entries if the background worker crashes."

**For non-critical activity feeds (connection to Week 11 — Celery):**

```python
# tasks/audit.py
from app.worker import celery

@celery.task(
    autoretry_for=(Exception,),
    retry_backoff=True,
    max_retries=3,
)
def record_audit_event(
    org_id: str,
    user_id: str,
    action: str,
    entity_type: str,
    entity_id: str,
    changes: dict | None = None,
    ip_address: str | None = None,
):
    """
    Background audit logging for non-critical events.
    
    Use for: page views, search queries, report generation
    Don't use for: data mutations (use synchronous for those)
    """
    # Celery tasks lose the request context.
    # That's why we pass org_id explicitly — no X-Org-ID header here.
    with get_sync_session() as session:
        entry = AuditLog(
            org_id=uuid.UUID(org_id),
            user_id=uuid.UUID(user_id),
            action=action,
            entity_type=entity_type,
            entity_id=uuid.UUID(entity_id),
            changes=changes,
            ip_address=ip_address,
        )
        session.add(entry)
        session.commit()
```

```
┌─────────────────────────────────────────────────────────────────┐
│         SYNC VS ASYNC AUDIT LOGGING — WHEN TO USE WHICH         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SYNCHRONOUS (same transaction):                                │
│  ├─ Data mutations (create, update, delete)                     │
│  ├─ Security events (login, password change, role change)       │
│  ├─ Compliance-required events                                  │
│  └─ GUARANTEE: if action happened, log exists                   │
│                                                                 │
│  ASYNCHRONOUS (background task):                                │
│  ├─ Read events (view, search, export)                          │
│  ├─ Activity feeds (non-critical)                               │
│  ├─ Analytics events                                            │
│  └─ TRADEOFF: slightly faster, but log might be delayed/lost   │
│                                                                 │
│  FOR YOUR CAPSTONE: Use synchronous for all mutations.          │
│  It's simpler and correct. Optimize later if needed.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.6 Querying the Audit Trail

**Expose the audit log as a read-only API endpoint for org admins:**

```python
# schemas/audit.py
from pydantic import BaseModel
from datetime import datetime


class AuditLogResponse(BaseModel):
    id: uuid.UUID
    user_id: uuid.UUID
    action: str
    entity_type: str
    entity_id: uuid.UUID
    changes: dict | None
    ip_address: str | None
    created_at: datetime

    model_config = {"from_attributes": True}
```

```python
# routes/audit.py

@router.get("/audit-logs", response_model=list[AuditLogResponse])
async def list_audit_logs(
    entity_type: str | None = None,
    entity_id: uuid.UUID | None = None,
    membership: Membership = Depends(require_org_role(OrgRole.ADMIN)),
    session: AsyncSession = Depends(get_session),
):
    """
    List audit logs for the current organization.
    Only org admins can view the audit trail.
    """
    stmt = (
        select(AuditLog)
        .where(AuditLog.org_id == membership.org_id)
        .order_by(AuditLog.created_at.desc())
        .limit(100)
    )

    if entity_type:
        stmt = stmt.where(AuditLog.entity_type == entity_type)
    if entity_id:
        stmt = stmt.where(AuditLog.entity_id == entity_id)

    result = await session.execute(stmt)
    return result.scalars().all()
```

> "Notice: this endpoint ALSO filters by org_id. Even audit logs are tenant-scoped. An Acme admin cannot see Wayne Enterprises' audit trail. The mantra never stops: every query, every time."

---

# PART 6: SOFT DELETES & DATA RETENTION

## 6.1 Why DELETE is Dangerous

**You introduced soft deletes briefly in Week 5. Now let's see why they're essential for multi-tenant SaaS.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 HARD DELETE vs SOFT DELETE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HARD DELETE: DELETE FROM projects WHERE id = '7a...';          │
│                                                                 │
│  ├─ Row is GONE from the database. Permanently.                 │
│  ├─ Foreign keys cascade (tasks, comments → also gone)          │
│  ├─ Cannot be undone without a database backup                  │
│  ├─ Audit log says "deleted" but the DATA is lost              │
│  └─ If a bug triggers it: catastrophic data loss                │
│                                                                 │
│                                                                 │
│  SOFT DELETE: UPDATE projects SET deleted_at = NOW()             │
│              WHERE id = '7a...';                                │
│                                                                 │
│  ├─ Row is STILL in the database, marked as deleted             │
│  ├─ Can be restored: SET deleted_at = NULL                      │
│  ├─ Queries filter out: WHERE deleted_at IS NULL                │
│  ├─ Audit log + data = full recovery possible                  │
│  └─ If a bug triggers it: annoying but reversible               │
│                                                                 │
│  For SaaS: soft delete is the default. Hard delete is the       │
│  exception (scheduled cleanup, GDPR "right to be forgotten").   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Hard delete is like shredding a document. Soft delete is like filing it in a drawer labeled 'archived.' You can always pull it back out."

---

## 6.2 The Soft Delete Mixin

```python
# models/mixins.py (add to your existing mixins)
from datetime import datetime

from sqlalchemy.orm import Mapped, mapped_column


class SoftDeleteMixin:
    """
    Add to any model that should support soft deletion.
    
    Usage:
        class Project(Base, TenantMixin, SoftDeleteMixin, TimestampMixin):
            ...
    
    Soft-deleted records have deleted_at set to a timestamp.
    Active records have deleted_at = None.
    """

    deleted_at: Mapped[datetime | None] = mapped_column(
        default=None,
        index=True,  # We filter on this in EVERY query
    )

    @property
    def is_deleted(self) -> bool:
        return self.deleted_at is not None
```

**Apply it to your models:**

```python
class Project(Base, TenantMixin, SoftDeleteMixin, TimestampMixin):
    __tablename__ = "projects"
    # Now has: id, org_id, deleted_at, created_at, updated_at
    # Plus all project-specific columns
    ...

class Task(Base, TenantMixin, SoftDeleteMixin, TimestampMixin):
    __tablename__ = "tasks"
    ...
```

---

## 6.3 Filtering by Default

**The most dangerous thing about soft deletes: forgetting to filter them out.**

> "If you soft-delete a project but your `GET /projects` endpoint doesn't filter `WHERE deleted_at IS NULL`, the 'deleted' project keeps appearing. That's confusing at best, a data leak at worst."

**Update the base repository to filter automatically:**

```python
# repositories/base.py (updated)

class TenantRepository(Generic[T]):

    def __init__(
        self,
        model: type[T],
        session: AsyncSession,
        org_id: uuid.UUID,
    ):
        self.model = model
        self.session = session
        self.org_id = org_id
        # Detect at init time if this model supports soft deletes
        self._has_soft_delete = hasattr(model, "deleted_at")

    def _base_query(self) -> Select:
        """
        ALWAYS scoped by org_id.
        AUTOMATICALLY filters out soft-deleted records.
        """
        stmt = select(self.model).where(
            self.model.org_id == self.org_id
        )

        if self._has_soft_delete:
            stmt = stmt.where(self.model.deleted_at.is_(None))

        return stmt

    # --- New soft-delete methods ---

    async def soft_delete(self, entity_id: uuid.UUID) -> T | None:
        """Mark a record as deleted without removing it."""
        entity = await self.get_by_id(entity_id)
        if entity is None:
            return None

        entity.deleted_at = datetime.utcnow()
        await self.session.flush()
        return entity

    async def restore(self, entity_id: uuid.UUID) -> T | None:
        """Restore a soft-deleted record."""
        # Bypass the normal _base_query (which excludes deleted)
        stmt = (
            select(self.model)
            .where(
                self.model.org_id == self.org_id,
                self.model.id == entity_id,
                self.model.deleted_at.is_not(None),  # Must BE deleted
            )
        )
        result = await self.session.execute(stmt)
        entity = result.scalar_one_or_none()

        if entity is None:
            return None

        entity.deleted_at = None
        await self.session.flush()
        return entity

    async def get_all_including_deleted(self) -> Sequence[T]:
        """
        Admin-only: bypass the soft-delete filter.
        Use for trash/archive views.
        """
        stmt = select(self.model).where(
            self.model.org_id == self.org_id
        )
        # No deleted_at filter — intentionally
        result = await self.session.execute(stmt)
        return result.scalars().all()
```

**Visualize the query behavior:**

```
┌─────────────────────────────────────────────────────────────────┐
│            SOFT DELETE QUERY BEHAVIOR                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Database state:                                                │
│  ┌──────────┬──────────────────┬─────────────────────┐          │
│  │ org_id   │ name             │ deleted_at           │          │
│  ├──────────┼──────────────────┼─────────────────────┤          │
│  │ acme     │ Website Redesign │ NULL                 │  Active  │
│  │ acme     │ Q4 Plan          │ 2026-02-10 14:30:00 │  Deleted │
│  │ acme     │ Mobile App       │ NULL                 │  Active  │
│  └──────────┴──────────────────┴─────────────────────┘          │
│                                                                 │
│                                                                 │
│  repo.get_all()                                                 │
│  → WHERE org_id = 'acme' AND deleted_at IS NULL                 │
│  → Returns: [Website Redesign, Mobile App]                      │
│    (Q4 Plan is hidden — correct!)                               │
│                                                                 │
│                                                                 │
│  repo.get_all_including_deleted()                               │
│  → WHERE org_id = 'acme'                                        │
│  → Returns: [Website Redesign, Q4 Plan, Mobile App]             │
│    (Admin trash view — shows everything)                        │
│                                                                 │
│                                                                 │
│  repo.restore(q4_plan_id)                                       │
│  → SET deleted_at = NULL WHERE id = ... AND deleted_at IS NOT   │
│    NULL                                                         │
│  → Q4 Plan is back!                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.4 Cascading Soft Deletes

**When you soft-delete a project, should its tasks also be soft-deleted?**

```
┌─────────────────────────────────────────────────────────────────┐
│            CASCADE SOFT DELETE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPTION A: Don't cascade                                        │
│  ├─ Soft-delete the project only                                │
│  ├─ Tasks still exist, but their parent is "deleted"            │
│  ├─ Problem: orphaned tasks show up in task lists               │
│  └─ Need to filter tasks where project.deleted_at IS NULL       │
│                                                                 │
│  OPTION B: Cascade soft deletes manually                        │
│  ├─ Soft-delete the project AND all its tasks                   │
│  ├─ Restore the project AND all its tasks                       │
│  ├─ Clean, predictable behavior                                 │
│  └─ More code, but correct UX                                   │
│                                                                 │
│  We go with OPTION B.                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# repositories/project.py (updated)

class ProjectRepository(TenantRepository[Project]):

    def __init__(self, session: AsyncSession, org_id: uuid.UUID):
        super().__init__(Project, session, org_id)

    async def soft_delete_with_cascade(
        self,
        project_id: uuid.UUID,
    ) -> Project | None:
        """Soft-delete a project AND all its tasks."""

        project = await self.get_by_id(project_id)
        if project is None:
            return None

        now = datetime.utcnow()

        # Soft-delete the project
        project.deleted_at = now

        # Cascade: soft-delete all tasks belonging to this project
        stmt = (
            update(Task)
            .where(
                Task.project_id == project_id,
                Task.org_id == self.org_id,      # Belt AND suspenders
                Task.deleted_at.is_(None),        # Only active tasks
            )
            .values(deleted_at=now)
        )
        await self.session.execute(stmt)
        await self.session.flush()

        return project

    async def restore_with_cascade(
        self,
        project_id: uuid.UUID,
    ) -> Project | None:
        """Restore a project AND all tasks deleted at the same time."""

        # Find the deleted project
        stmt = (
            select(Project)
            .where(
                Project.id == project_id,
                Project.org_id == self.org_id,
                Project.deleted_at.is_not(None),
            )
        )
        result = await self.session.execute(stmt)
        project = result.scalar_one_or_none()

        if project is None:
            return None

        deleted_at = project.deleted_at  # Save the timestamp

        # Restore the project
        project.deleted_at = None

        # Cascade restore: tasks deleted at the SAME timestamp
        # (This avoids restoring tasks that were individually deleted
        #  BEFORE the project was deleted)
        stmt = (
            update(Task)
            .where(
                Task.project_id == project_id,
                Task.org_id == self.org_id,
                Task.deleted_at == deleted_at,  # Same cascade timestamp
            )
            .values(deleted_at=None)
        )
        await self.session.execute(stmt)
        await self.session.flush()

        return project
```

> "Notice the `Task.deleted_at == deleted_at` check in the restore. This is subtle but important. If Bob manually deleted Task #5 on Monday, and the entire project was soft-deleted on Wednesday, restoring the project on Thursday should NOT bring back Task #5. Only the tasks that were cascade-deleted alongside the project should be restored. The matching timestamp is the signal."

---

## 6.5 Hard Delete Policies (Scheduled Cleanup)

**Soft-deleted records accumulate. Eventually, you must truly remove them.**

```
┌─────────────────────────────────────────────────────────────────┐
│              DATA RETENTION POLICY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Soft-deleted for < 30 days:                                    │
│  └─ Restorable. Visible in "Trash" view.                        │
│                                                                 │
│  Soft-deleted for 30-90 days:                                   │
│  └─ Not shown in UI. Still in database.                         │
│     Can be restored by support request.                         │
│                                                                 │
│  Soft-deleted for > 90 days:                                    │
│  └─ HARD DELETE. Permanently removed.                           │
│     Scheduled cleanup job runs nightly.                         │
│                                                                 │
│  These numbers are YOUR policy decision.                        │
│  Document them. ADR-005: "Data retention policy."               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The cleanup task (connection to Week 11 — Celery Beat):**

```python
# tasks/cleanup.py
from datetime import datetime, timedelta

from sqlalchemy import delete

from app.worker import celery
from app.database import get_sync_session
from app.models.task import Task
from app.models.project import Project


@celery.task
def hard_delete_expired_records():
    """
    Permanently delete records soft-deleted more than 90 days ago.
    
    Scheduled via Celery Beat to run nightly at 3 AM UTC.
    """
    cutoff = datetime.utcnow() - timedelta(days=90)

    with get_sync_session() as session:
        # Delete tasks first (foreign key to projects)
        task_result = session.execute(
            delete(Task).where(
                Task.deleted_at.is_not(None),
                Task.deleted_at < cutoff,
            )
        )

        # Then delete projects
        project_result = session.execute(
            delete(Project).where(
                Project.deleted_at.is_not(None),
                Project.deleted_at < cutoff,
            )
        )

        session.commit()

        return {
            "tasks_deleted": task_result.rowcount,
            "projects_deleted": project_result.rowcount,
        }


# Celery Beat schedule (in your celery config):
# beat_schedule = {
#     "hard-delete-expired": {
#         "task": "app.tasks.cleanup.hard_delete_expired_records",
#         "schedule": crontab(hour=3, minute=0),  # 3 AM UTC daily
#     },
# }
```

> "Notice: this task does NOT filter by org_id. It cleans up across ALL tenants. This is correct — it's a system-level maintenance job, not a tenant operation. It runs in a background worker, not in a request context."

**GDPR "Right to Be Forgotten" (awareness):**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  GDPR Article 17 may require you to hard-delete a specific      │
│  user's data ON REQUEST, regardless of your retention policy.   │
│                                                                 │
│  This is a separate code path from soft-delete:                 │
│  ├─ Find all data belonging to the user                         │
│  ├─ Anonymize or hard-delete it                                 │
│  ├─ Log that you did it (ironically, you audit the deletion)    │
│  └─ Respond to the user confirming completion                   │
│                                                                 │
│  We won't implement this — but know it exists.                  │
│  For your capstone, soft delete with scheduled cleanup          │
│  is sufficient.                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│           MULTI-TENANCY QUICK REFERENCE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TENANT MODEL:                                                  │
│      Organization → Membership (with role) ← User               │
│      Many-to-many with OrgRole per membership                   │
│                                                                 │
│  TENANT MIXIN:                                                  │
│      class TenantMixin:                                         │
│          @declared_attr                                         │
│          def org_id(cls) -> Mapped[uuid.UUID]:                  │
│              return mapped_column(ForeignKey("organizations.id"))│
│                                                                 │
│  TENANT CONTEXT:                                                │
│      X-Org-ID header → get_current_org() dependency             │
│      Validates user membership in DB every request              │
│                                                                 │
│  TENANT REPOSITORY:                                             │
│      _base_query() adds WHERE org_id = :current_org             │
│      + WHERE deleted_at IS NULL (if soft-delete model)          │
│      create() injects org_id automatically                      │
│                                                                 │
│  AUDIT LOGGING:                                                 │
│      Append-only table: who, what, which, when, how             │
│      changes column: JSONB diff of old vs new values            │
│      Synchronous for mutations, async for reads                 │
│                                                                 │
│  SOFT DELETE:                                                   │
│      deleted_at: datetime | None  (NULL = active)               │
│      Cascade via matching timestamp                             │
│      Hard delete via scheduled Celery task                      │
│                                                                 │
│  THE MANTRA:                                                    │
│      Every query. Every time. No exceptions.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  MULTI-TENANCY = INVISIBLE WALLS                                │
│                                                                 │
│  Your database has ONE set of tables. Multiple organizations    │
│  store their data side by side. The org_id column + your        │
│  repository layer creates invisible walls between them.         │
│                                                                 │
│  ┌──────────────────────────────────────────────┐               │
│  │              projects table                  │               │
│  │  ┌─────────────┬─┬──────────────┬─┬────────┐ │               │
│  │  │  ACME data  │█│  WAYNE data  │█│ STARK  │ │               │
│  │  │             │█│              │█│  data  │ │               │
│  │  │  org_id=A   │█│  org_id=W    │█│org_id=S│ │               │
│  │  └─────────────┘█└──────────────┘█└────────┘ │               │
│  │                 █                █           │               │
│  │           INVISIBLE WALLS (WHERE clauses)    │               │
│  └──────────────────────────────────────────────┘               │
│                                                                 │
│  THE APARTMENT BUILDING:                                        │
│  ├─ Organization    = The tenant (Apartment 301)                │
│  ├─ Membership      = The lease (who has keys)                  │
│  ├─ OrgRole         = The permissions (owner vs guest)          │
│  ├─ TenantMixin     = The apartment number on every piece       │
│  │                    of furniture                              │
│  ├─ get_current_org = The front desk checking your keycard      │
│  ├─ _base_query     = The lock on every door                    │
│  ├─ Audit log       = The security camera footage               │
│  └─ Soft delete     = Moving to storage instead of the dump     │
│                                                                 │
│  EVERY QUERY. EVERY TIME. NO EXCEPTIONS.                        │
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
│  WEEK 13, LECTURE 3: Integration Patterns                       │
│  └─ Email notifications, file uploads, search —                 │
│     all must be tenant-scoped. S3 paths include org_id.         │
│     Search indexes filter by org_id.                            │
│                                                                 │
│  WEEK 13, LECTURE 4: Code Review & Technical Communication      │
│  └─ Your code review checklist now includes:                    │
│     "Does every query go through a tenant-scoped repo?"         │
│     "Are audit log calls present for all mutations?"            │
│                                                                 │
│  WEEK 13-14 CAPSTONE PROJECT:                                   │
│  └─ You're building a Multi-Tenant SaaS Backend.                │
│     Everything in this lecture is your implementation plan.      │
│     TenantMixin, get_current_org, scoped repos,                 │
│     audit logs, soft deletes — all required.                    │
│                                                                 │
│  WEEK 15: Docker & CI/CD                                        │
│  └─ Alembic migrations in CI must handle the org_id columns.    │
│     Your test pipeline must include tenant isolation tests.     │
│                                                                 │
│  WEEK 16: System Design                                         │
│  └─ "Design a multi-tenant SaaS platform" is a classic          │
│     system design interview question. You've now built one.     │
│     You can draw the architecture from experience, not          │
│     from memorized diagrams.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```