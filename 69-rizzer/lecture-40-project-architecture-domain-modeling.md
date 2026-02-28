# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PAIN FIRST, PLANNING SECOND                                    │
│  ───────────────────────────                                    │
│  Students must SEE the chaos of an unplanned large project      │
│  before they'll accept the discipline of architecture.          │
│  We'll show them the mess. Let them cringe.                     │
│                                                                 │
│  DERIVE, DON'T DICTATE                                          │
│  ─────────────────────                                          │
│  The folder structure is not a template to memorize.            │
│  It FOLLOWS from the domain model, which FOLLOWS from          │
│  requirements. Students discover the structure, not copy it.    │
│                                                                 │
│  CONNECT THE 12-WEEK JOURNEY                                    │
│  ───────────────────────────                                    │
│  Repository pattern (Week 6), Pydantic schemas (Week 3),       │
│  dependency injection (Week 3), auth layers (Week 9) —          │
│  they've been building pieces of architecture for weeks.        │
│  This lecture reveals the full picture.                         │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Building a shed vs. building a house. You've been building     │
│  sheds. Now the scale changes. You need blueprints.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│              PROJECT ARCHITECTURE & DOMAIN MODELING              │
│                     (3.5-4 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (35 min)                                   │
│  ├─ 1.1 The "Just Start Coding" Demo                            │
│  ├─ 1.2 Five Ways This Falls Apart                              │
│  ├─ 1.3 You've Already Felt This                                │
│  └─ 1.4 The Blueprint Analogy                                   │
│                                                                 │
│  PART 2: REQUIREMENTS ANALYSIS (40 min)                         │
│  ├─ 2.1 The Business Brief (Your Starting Point)                │
│  ├─ 2.2 Extracting Nouns and Verbs                              │
│  ├─ 2.3 User Stories — The Structured Format                    │
│  ├─ 2.4 Functional vs Non-Functional Requirements               │
│  └─ 2.5 Prioritization: Must / Should / Could                   │
│                                                                 │
│  PART 3: DOMAIN MODELING (60 min)                               │
│  ├─ 3.1 What Is a Domain Model? (Not a Database Schema)         │
│  ├─ 3.2 Step 1 — Identifying Entities                           │
│  ├─ 3.3 Step 2 — Defining Attributes                            │
│  ├─ 3.4 Step 3 — Mapping Relationships                          │
│  ├─ 3.5 Step 4 — Grouping into Modules (Boundaries)             │
│  └─ 3.6 From Domain Model to ER Diagram                         │
│                                                                 │
│  PART 4: PROJECT STRUCTURE (45 min)                             │
│  ├─ 4.1 The Layered Architecture                                │
│  ├─ 4.2 Request Flow Through Layers                             │
│  ├─ 4.3 The Dependency Rule                                     │
│  ├─ 4.4 The Full Folder Layout                                  │
│  └─ 4.5 Module Isolation — One Domain Per Package               │
│                                                                 │
│  PART 5: ARCHITECTURE DECISION RECORDS (25 min)                 │
│  ├─ 5.1 Why Document Decisions?                                 │
│  ├─ 5.2 The ADR Template                                        │
│  └─ 5.3 Two Real ADRs for the Capstone                          │
│                                                                 │
│  PART 6: API CONTRACT FIRST (30 min)                            │
│  ├─ 6.1 Design Before Implementation                            │
│  ├─ 6.2 Schema-First with Pydantic                              │
│  ├─ 6.3 Router Stubs — The Skeleton                             │
│  └─ 6.4 From Contract to Implementation Plan                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The "Just Start Coding" Demo

**Start by doing what most students would do. Open a single file and start building the capstone.**

```python
# main.py — "I'll just start coding and figure it out"

from fastapi import FastAPI, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import (
    AsyncSession, create_async_engine, async_sessionmaker
)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import String, ForeignKey, select, Boolean, DateTime
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime, timezone
from passlib.context import CryptContext
from jose import jwt
from typing import Optional
import redis.asyncio as redis
from celery import Celery

app = FastAPI()

# --- Database setup right here ---
engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/saas")
async_session = async_sessionmaker(engine)

# --- Redis right here ---
redis_client = redis.from_url("redis://localhost")

# --- Celery right here ---
celery_app = Celery("tasks", broker="redis://localhost")

# --- Password hashing right here ---
pwd_context = CryptContext(schemes=["bcrypt"])

# --- SQLAlchemy Models right here ---
class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True)
    hashed_password: Mapped[str] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(default=True)
    memberships: Mapped[list["Membership"]] = relationship(back_populates="user")

class Organization(Base):
    __tablename__ = "organizations"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    slug: Mapped[str] = mapped_column(String(100), unique=True)
    memberships: Mapped[list["Membership"]] = relationship(back_populates="org")
    projects: Mapped[list["Project"]] = relationship(back_populates="org")

class Membership(Base):
    __tablename__ = "memberships"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    org_id: Mapped[int] = mapped_column(ForeignKey("organizations.id"))
    role: Mapped[str] = mapped_column(String(20))  # "admin", "member", "viewer"
    user: Mapped["User"] = relationship(back_populates="memberships")
    org: Mapped["Organization"] = relationship(back_populates="memberships")

class Project(Base):
    __tablename__ = "projects"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    org_id: Mapped[int] = mapped_column(ForeignKey("organizations.id"))
    org: Mapped["Organization"] = relationship(back_populates="projects")
    tasks: Mapped[list["Task"]] = relationship(back_populates="project")

class Task(Base):
    __tablename__ = "tasks"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(300))
    description: Mapped[Optional[str]]
    status: Mapped[str] = mapped_column(String(20), default="todo")
    assignee_id: Mapped[Optional[int]] = mapped_column(ForeignKey("users.id"))
    project_id: Mapped[int] = mapped_column(ForeignKey("projects.id"))
    project: Mapped["Project"] = relationship(back_populates="tasks")
    comments: Mapped[list["Comment"]] = relationship(back_populates="task")

class Comment(Base):
    __tablename__ = "comments"
    id: Mapped[int] = mapped_column(primary_key=True)
    content: Mapped[str]
    task_id: Mapped[int] = mapped_column(ForeignKey("tasks.id"))
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    task: Mapped["Task"] = relationship(back_populates="comments")

class AuditLog(Base):
    __tablename__ = "audit_logs"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    action: Mapped[str] = mapped_column(String(50))
    entity_type: Mapped[str] = mapped_column(String(50))
    entity_id: Mapped[int]
    timestamp: Mapped[datetime] = mapped_column(default_factory=lambda: datetime.now(timezone.utc))


# --- Pydantic Schemas right here ---
class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8)

class UserResponse(BaseModel):
    id: int
    email: str
    is_active: bool
    model_config = {"from_attributes": True}

class OrgCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    slug: str = Field(pattern=r"^[a-z0-9-]+$")

class OrgResponse(BaseModel):
    id: int
    name: str
    slug: str
    model_config = {"from_attributes": True}

class ProjectCreate(BaseModel):
    name: str = Field(min_length=1, max_length=200)

class ProjectResponse(BaseModel):
    id: int
    name: str
    org_id: int
    model_config = {"from_attributes": True}

class TaskCreate(BaseModel):
    title: str = Field(min_length=1, max_length=300)
    description: Optional[str] = None
    assignee_id: Optional[int] = None

class TaskResponse(BaseModel):
    id: int
    title: str
    status: str
    assignee_id: Optional[int]
    project_id: int
    model_config = {"from_attributes": True}

class CommentCreate(BaseModel):
    content: str = Field(min_length=1)

# ... 12 more schemas for updates, filters, pagination ...


# --- Dependency right here ---
async def get_db():
    async with async_session() as session:
        yield session

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    # 15 lines of JWT decode, DB lookup, error handling...
    ...

async def require_org_admin(
    org_id: int,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> User:
    # 12 lines of membership check, role check...
    ...


# --- Auth Routes right here ---
@app.post("/auth/register", response_model=UserResponse)
async def register(data: UserCreate, db: AsyncSession = Depends(get_db)):
    # Check duplicate email
    result = await db.execute(select(User).where(User.email == data.email))
    if result.scalar_one_or_none():
        raise HTTPException(status_code=409, detail="Email taken")
    hashed = pwd_context.hash(data.password)
    user = User(email=data.email, hashed_password=hashed)
    db.add(user)
    await db.commit()
    await db.refresh(user)
    # Log to audit
    log = AuditLog(user_id=user.id, action="register", entity_type="user", entity_id=user.id)
    db.add(log)
    await db.commit()
    # Send welcome email via Celery
    celery_app.send_task("send_welcome_email", args=[user.email])
    return user

@app.post("/auth/login")
async def login(data: UserCreate, db: AsyncSession = Depends(get_db)):
    # 20 lines of credential verification, token generation...
    ...

# --- Organization Routes right here ---
@app.post("/orgs", response_model=OrgResponse)
async def create_org(
    data: OrgCreate,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    org = Organization(name=data.name, slug=data.slug)
    db.add(org)
    await db.flush()
    membership = Membership(user_id=user.id, org_id=org.id, role="admin")
    db.add(membership)
    log = AuditLog(user_id=user.id, action="create", entity_type="organization", entity_id=org.id)
    db.add(log)
    await db.commit()
    await db.refresh(org)
    # Invalidate org cache
    await redis_client.delete(f"user:{user.id}:orgs")
    return org

# --- Project Routes ---
# @app.post("/orgs/{org_id}/projects", ...)
# @app.get("/orgs/{org_id}/projects", ...)
# @app.get("/orgs/{org_id}/projects/{project_id}", ...)
# @app.put("/orgs/{org_id}/projects/{project_id}", ...)
# @app.delete("/orgs/{org_id}/projects/{project_id}", ...)

# --- Task Routes ---
# @app.post("/orgs/{org_id}/projects/{project_id}/tasks", ...)
# @app.get("/orgs/{org_id}/projects/{project_id}/tasks", ...)  + filtering + pagination
# @app.get("/orgs/{org_id}/projects/{project_id}/tasks/{task_id}", ...)
# @app.patch("/orgs/{org_id}/projects/{project_id}/tasks/{task_id}", ...)
# @app.delete("/orgs/{org_id}/projects/{project_id}/tasks/{task_id}", ...)

# --- Comment Routes ---
# @app.post("/.../tasks/{task_id}/comments", ...)
# @app.get("/.../tasks/{task_id}/comments", ...)
# @app.delete("/.../comments/{comment_id}", ...)

# --- Member Management Routes ---
# @app.post("/orgs/{org_id}/members", ...)
# @app.get("/orgs/{org_id}/members", ...)
# @app.patch("/orgs/{org_id}/members/{member_id}", ...)
# @app.delete("/orgs/{org_id}/members/{member_id}", ...)

# --- Search, Admin, Audit Log, Notification Routes ---
# ... 15+ more endpoints ...

# --- WebSocket for real-time notifications ---
# @app.websocket("/ws/notifications")
# async def notifications_ws(...): ...

# And we haven't even added:
# - File attachments
# - Report generation (background task)
# - Cache invalidation logic per route
# - Rate limiting per endpoint group
# - ...
```

**Pause. Look at the students.**

> "We've written ~200 lines and we've only implemented TWO of about 35 endpoints. We haven't implemented most route bodies, we have no tests, no migrations, and this file is ALREADY hard to read. How long is this file going to be when we're done?"

Answer: **Conservatively 3,000–4,000 lines. In one file.**

---

## 1.2 Five Ways This Falls Apart

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY THE SINGLE-FILE APPROACH FAILS                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ① FINDING ANYTHING IS A NIGHTMARE                              │
│     "Where's the task status update logic?"                     │
│     Answer: Somewhere around line 1,847. Good luck.             │
│     You spend more time SEARCHING than CODING.                  │
│                                                                 │
│  ② CHANGING ONE THING BREAKS ANOTHER                            │
│     You rename a User field. Now you have to update:            │
│     the model, 4 schemas, 6 route handlers, 3 dependencies,    │
│     and the audit log format. All in the same file.             │
│     Miss one? Runtime crash in production.                      │
│                                                                 │
│  ③ COLLABORATION IS IMPOSSIBLE                                  │
│     You work on tasks. Your teammate works on comments.         │
│     Both of you edit main.py.                                   │
│     Git merge conflict on a 3,000-line file. Every. Time.       │
│                                                                 │
│  ④ TESTING BECOMES TORTURE                                      │
│     Want to test task creation?                                 │
│     The logic is tangled with auth, database, caching,          │
│     audit logging, and Celery calls. You can't isolate          │
│     the thing you want to test from everything else.            │
│                                                                 │
│  ⑤ YOUR BRAIN CAN'T HOLD IT                                    │
│     Humans can hold ~7 things in working memory.                │
│     This file asks you to hold: models, schemas, auth,         │
│     dependencies, caching, background jobs, WebSockets,         │
│     audit logging, validation, error handling...                │
│     all at once. You WILL make mistakes.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 You've Already Felt This

**Don't lecture them — remind them of their experience:**

> "Think back to what happened with your Task Manager over the last 10 weeks."

```
┌─────────────────────────────────────────────────────────────────┐
│                 YOUR TASK MANAGER JOURNEY                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Week 3-4: In-memory Task Manager                               │
│  ├─ A few files, a few endpoints                                │
│  ├─ Simple, fast, easy to understand                            │
│  └─ You could hold the ENTIRE project in your head              │
│                                                                 │
│  Week 6: Added PostgreSQL + SQLAlchemy + Alembic                │
│  ├─ Suddenly you had models/ AND schemas/                       │
│  ├─ You introduced the repository pattern                       │
│  ├─ The project doubled in size                                 │
│  └─ Still manageable, but growing                               │
│                                                                 │
│  Week 9: Added Auth + RBAC                                      │
│  ├─ JWT logic, password hashing, role checking                  │
│  ├─ New dependencies, new middleware                            │
│  ├─ "Where does auth code go?"                                  │
│  └─ Starting to feel messy?                                     │
│                                                                 │
│  Week 10: Added Redis caching                                   │
│  ├─ Cache logic woven into existing routes                      │
│  └─ "Do I put cache invalidation in the route or repository?"   │
│                                                                 │
│  Week 11: Added Celery background jobs                          │
│  ├─ Worker definitions, task triggers from routes               │
│  └─ "This project has a LOT of moving parts now..."             │
│                                                                 │
│  Week 12: Added WebSockets + performance optimization           │
│  ├─ Connection manager, broadcast logic                         │
│  └─ "I keep losing track of what depends on what."              │
│                                                                 │
│  ───────────────────────────────────────────────────────────     │
│  That was ONE domain entity (tasks) with add-on features.       │
│  The capstone has 8-10 entities, multi-tenancy, AND all         │
│  those same features. Scale matters.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "At which point in that journey did you start wishing you had planned your structure better BEFORE coding?"

---

## 1.4 The Blueprint Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE BLUEPRINT ANALOGY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BUILDING A SHED                                                │
│  ───────────────                                                │
│  • One room, one purpose                                        │
│  • Build in a weekend                                           │
│  • No blueprint needed — it fits in your head                   │
│  • If the wall is wrong, tear it down, rebuild                  │
│  • This is your Task Manager in Week 3.                         │
│                                                                 │
│                                                                 │
│  BUILDING A HOUSE                                               │
│  ────────────────                                               │
│  • Multiple rooms with different purposes                       │
│  • Plumbing, electrical, heating must all work together         │
│  • You NEED blueprints — too complex for one brain              │
│  • If the FOUNDATION is wrong, you can't easily fix it          │
│  • Different contractors work on different systems               │
│  • Inspector checks compliance before you move in               │
│  • This is your Capstone.                                       │
│                                                                 │
│                                                                 │
│  MAPPING:                                                       │
│  ────────                                                       │
│  Blueprints               →  Architecture                       │
│  "What rooms do we need?" →  Requirements Analysis              │
│  Room layout & flow       →  Domain Modeling                    │
│  Floor plan               →  Project Structure                  │
│  Building code records    →  Architecture Decision Records      │
│  Permit documents         →  API Contract                       │
│                                                                 │
│  You cannot build a house by starting with the kitchen sink.    │
│  You start with the blueprint.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The pipeline we'll follow for the rest of this lecture:**

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Business   │    │   Domain     │    │   Project    │
│   Brief      │───▶│   Model      │───▶│   Structure  │
│              │    │              │    │              │
│  (what the   │    │ (entities,   │    │ (folders,    │
│   client     │    │  relations,  │    │  layers,     │
│   wants)     │    │  boundaries) │    │  modules)    │
└──────────────┘    └──────────────┘    └──────┬───────┘
                                               │
                    ┌──────────────┐    ┌───────▼──────┐
                    │   ADRs       │    │   API        │
                    │              │◀───│   Contract   │
                    │  (why we     │    │              │
                    │   chose      │    │ (endpoints,  │
                    │   this way)  │    │  schemas)    │
                    └──────────────┘    └──────────────┘

Each step produces an ARTIFACT. Each artifact feeds the next.
You don't write code until ALL of these exist.
```

---

# PART 2: REQUIREMENTS ANALYSIS

## 2.1 The Business Brief (Your Starting Point)

**Every real project starts with a business need, not a technology choice.**

> "Imagine a client sends you this email. This is the kind of thing you receive in the real world. It's messy, informal, and incomplete. Your job is to turn it into something you can build."

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE CLIENT EMAIL                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Subject: Project management tool — backend                     │
│                                                                 │
│  Hey,                                                           │
│                                                                 │
│  We need a backend for a project management tool. Think of      │
│  a simplified Jira / Asana.                                     │
│                                                                 │
│  Different companies sign up and each company manages their     │
│  own projects and teams. Each company should only see their     │
│  own data. Users can belong to multiple companies.              │
│                                                                 │
│  Within a company, there are projects. Each project has tasks.  │
│  Tasks can be assigned to people, have statuses (todo, in       │
│  progress, done), priorities, and due dates. People should be   │
│  able to comment on tasks.                                      │
│                                                                 │
│  We need roles — some people are admins who can manage          │
│  everything, regular members who can create and edit tasks,     │
│  and viewers who can only read. Would be great if there were    │
│  real-time notifications when someone assigns you a task or     │
│  comments on yours.                                             │
│                                                                 │
│  Also need an audit trail — who changed what and when.          │
│  We might want to export data later (reports, CSV).             │
│  Definitely need search so people can find tasks.               │
│                                                                 │
│  Oh, and file attachments on tasks would be really nice.        │
│                                                                 │
│  We're expecting ~50 companies using it within the first year,  │
│  probably 500 total users.                                      │
│                                                                 │
│  Can you build the API? We'll handle the frontend.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "You just read that email. What's the FIRST thing you do? Open your editor and start a FastAPI project?"

Answer: **No. You read it again. With a pen.**

---

## 2.2 Extracting Nouns and Verbs

**This is a technique, not intuition. You do this mechanically.**

> "Go through the brief. Circle every NOUN (these are candidate entities). Underline every VERB (these are candidate behaviors/operations)."

```
┌─────────────────────────────────────────────────────────────────┐
│                   NOUN EXTRACTION                               │
│              (Candidate Entities)                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  From the brief, the nouns:                                     │
│                                                                 │
│  • companies (organizations)    ← ENTITY (has identity)         │
│  • users (people)               ← ENTITY                       │
│  • projects                     ← ENTITY                       │
│  • tasks                        ← ENTITY                       │
│  • statuses                     ← ATTRIBUTE of Task             │
│  • priorities                   ← ATTRIBUTE of Task             │
│  • due dates                    ← ATTRIBUTE of Task             │
│  • comments                     ← ENTITY (has identity)         │
│  • roles (admin/member/viewer)  ← ATTRIBUTE of Membership       │
│  • notifications                ← ENTITY                       │
│  • audit trail                  ← ENTITY                       │
│  • reports / CSV export         ← OPERATION (not entity)        │
│  • file attachments             ← ENTITY                       │
│  • teams                        ← membership relationship       │
│                                                                 │
│  DECISION: What's an entity vs. an attribute?                   │
│  ──────────────────────────────────────────────                 │
│  Entity = has its OWN identity and lifecycle.                   │
│           You can create, read, update, delete it.              │
│           "Can I refer to it by an ID?"                         │
│                                                                 │
│  Attribute = a property OF an entity.                           │
│              No independent existence.                          │
│              "Does this make sense without its parent?"         │
│                                                                 │
│  "status" is not an entity — it's a property of a task.         │
│  "comment" IS an entity — it has its own creation time,         │
│   its own author, and you can delete a comment independently.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   VERB EXTRACTION                               │
│              (Candidate Operations)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  From the brief, the verbs:                                     │
│                                                                 │
│  • sign up                 → User registration                  │
│  • manage (projects/teams) → CRUD operations                    │
│  • see their own data      → Data isolation (multi-tenancy!)    │
│  • belong to (multiple)    → Many-to-many relationship          │
│  • assign (to people)      → Update + Notification trigger      │
│  • comment on              → Create comment                     │
│  • manage everything       → Admin permissions                  │
│  • create and edit         → Member permissions                 │
│  • only read               → Viewer permissions                 │
│  • notify                  → Real-time push (WebSocket)         │
│  • changed what and when   → Audit logging                      │
│  • export                  → Background job (CSV generation)    │
│  • search                  → Query with filters                 │
│  • attach                  → File upload                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The insight:**

> "You haven't written a single line of code, but you already have a rough picture of what you're building. The nouns become your database tables. The verbs become your endpoints and business logic. This is requirements analysis."

---

## 2.3 User Stories — The Structured Format

**Turn informal requirements into structured user stories:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     USER STORY FORMAT                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  As a [ROLE],                                                   │
│  I want to [ACTION],                                            │
│  so that [BENEFIT].                                             │
│                                                                 │
│  This format forces you to think about:                         │
│  • WHO is doing this (which role?)                              │
│  • WHAT they're doing (what's the operation?)                   │
│  • WHY they care (what's the value?)                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Translate the brief into stories:**

```
┌─────────────────────────────────────────────────────────────────┐
│                SAMPLE USER STORIES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AUTHENTICATION                                                 │
│  As a new user, I want to register with my email and password,  │
│  so that I can access the platform.                             │
│                                                                 │
│  As a registered user, I want to log in and receive a token,    │
│  so that I can make authenticated requests.                     │
│                                                                 │
│  ORGANIZATION MANAGEMENT                                        │
│  As a user, I want to create an organization,                   │
│  so that my company has a workspace for our projects.           │
│                                                                 │
│  As an org admin, I want to invite users to my organization,    │
│  so that my team can collaborate.                               │
│                                                                 │
│  As an org admin, I want to assign roles to members,            │
│  so that I can control who can do what.                         │
│                                                                 │
│  PROJECT & TASK MANAGEMENT                                      │
│  As a member, I want to create projects in my organization,     │
│  so that I can group related tasks.                             │
│                                                                 │
│  As a member, I want to create tasks within a project,          │
│  so that I can track work items.                                │
│                                                                 │
│  As a member, I want to assign tasks to other members,          │
│  so that work is distributed clearly.                           │
│                                                                 │
│  As a viewer, I want to view tasks and comments,                │
│  so that I can follow project progress.                         │
│                                                                 │
│  REAL-TIME                                                      │
│  As a member, I want to receive a notification when someone     │
│  assigns me a task or comments on my task,                      │
│  so that I can respond quickly.                                 │
│                                                                 │
│  AUDIT & ADMIN                                                  │
│  As an org admin, I want to see an audit trail of all changes,  │
│  so that I know who did what and when.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice something: each user story implies a role, an endpoint, and a permission check. 'As an org admin, I want to assign roles' means: PATCH `/orgs/{org_id}/members/{member_id}` — and only admins can call it. The requirements are ALREADY shaping your API."

---

## 2.4 Functional vs Non-Functional Requirements

**Not everything the client wants is a feature. Some things are CONSTRAINTS.**

```
┌─────────────────────────────────────────────────────────────────┐
│           FUNCTIONAL VS NON-FUNCTIONAL REQUIREMENTS             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FUNCTIONAL (What the system DOES)                              │
│  ──────────────────────────────────                             │
│  • User registration and login                                  │
│  • Create/read/update/delete tasks                              │
│  • Role-based access control                                    │
│  • Real-time notifications                                      │
│  • File attachments                                             │
│  • Search and filtering                                         │
│  • CSV export                                                   │
│                                                                 │
│  These become your ENDPOINTS and BUSINESS LOGIC.                │
│                                                                 │
│                                                                 │
│  NON-FUNCTIONAL (How the system BEHAVES)                        │
│  ────────────────────────────────────────                       │
│  • Data isolation: orgs MUST NOT see each other's data          │
│  • Performance: "~500 users" → modest scale                     │
│  • Security: passwords hashed, tokens expire, rate limiting     │
│  • Auditability: every mutation logged                          │
│  • Reliability: background jobs retry on failure                │
│                                                                 │
│  These become your ARCHITECTURE DECISIONS and MIDDLEWARE.        │
│                                                                 │
│                                                                 │
│  ⚠️  Non-functional requirements are INVISIBLE to the client    │
│  but they shape your architecture MORE than features do.        │
│                                                                 │
│  "Data isolation" alone dictates multi-tenancy strategy,        │
│  query filters on EVERY endpoint, test strategy, and more.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.5 Prioritization: Must / Should / Could

**You cannot build everything at once. Prioritize ruthlessly.**

```
┌─────────────────────────────────────────────────────────────────┐
│               MoSCoW PRIORITIZATION                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MUST HAVE (The system doesn't work without these)              │
│  ├─ User auth (registration, login, JWT)                        │
│  ├─ Organization CRUD + multi-tenancy (data isolation)          │
│  ├─ Project CRUD (within org)                                   │
│  ├─ Task CRUD (within project, with status, assignee)           │
│  ├─ Role-based access (admin, member, viewer)                   │
│  └─ Basic test suite                                            │
│                                                                 │
│  SHOULD HAVE (Expected, build right after core)                 │
│  ├─ Comments on tasks                                           │
│  ├─ Audit log                                                   │
│  ├─ Task filtering, sorting, pagination                         │
│  ├─ Caching for frequently read data                            │
│  └─ WebSocket notifications                                     │
│                                                                 │
│  COULD HAVE (Nice extras if time allows)                        │
│  ├─ File attachments                                            │
│  ├─ CSV export (background job)                                 │
│  ├─ Search (full-text)                                          │
│  └─ Admin statistics dashboard                                  │
│                                                                 │
│                                                                 │
│  BUILD ORDER: Must → Should → Could                             │
│                                                                 │
│  ⚠️  The architecture MUST support all three tiers,             │
│  even if you only BUILD the first tier initially.               │
│  This is why you design before you code.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key question to ask at this stage:**

> "Can I add file attachments LATER without rewriting my task system? If the answer is no, my architecture is wrong. The design should make future features ADDITIVE, not surgical."

---

# PART 3: DOMAIN MODELING

## 3.1 What Is a Domain Model? (Not a Database Schema)

```
┌─────────────────────────────────────────────────────────────────┐
│                    DOMAIN MODEL                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A domain model is a CONCEPTUAL MAP of the business problem.    │
│                                                                 │
│  It answers: "What are the THINGS in this system, and how       │
│  do they RELATE to each other?"                                 │
│                                                                 │
│  It is NOT:                                                     │
│  ├─ A database schema      (that's an implementation detail)    │
│  ├─ A class diagram        (that's a code detail)               │
│  ├─ An API endpoint list   (that's an interface detail)         │
│  └─ A file structure       (that's an organization detail)      │
│                                                                 │
│  It IS:                                                         │
│  ├─ A shared understanding of the problem space                 │
│  ├─ A vocabulary everyone (dev, PM, client) agrees on           │
│  ├─ A map of entities, their attributes, and relationships      │
│  └─ The FOUNDATION that everything else is derived from         │
│                                                                 │
│                                                                 │
│  Think of it this way:                                          │
│  ┌──────────┐                                                   │
│  │ Domain   │──── drives ────▶ Database schema                  │
│  │ Model    │──── drives ────▶ API endpoints                    │
│  │          │──── drives ────▶ Folder structure                 │
│  │          │──── drives ────▶ Test organization                │
│  └──────────┘                                                   │
│                                                                 │
│  Get the domain model wrong → everything downstream is wrong.   │
│  Get it right → everything else falls into place.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Back to the house analogy:**

> "The domain model is like deciding what rooms your house needs BEFORE drawing the floor plan. If you forget you need a laundry room, no floor plan will save you. You'll be running a dryer hose through the living room."

**Connection to Week 5:**

> "In Week 5, you learned ER diagrams — tables, columns, relationships, foreign keys. That was designing the DATABASE. A domain model sits one level ABOVE that. You first figure out the concepts, THEN translate to tables. Sometimes one concept becomes two tables. Sometimes two concepts share one table. The domain model is the thinking; the ER diagram is the implementation."

---

## 3.2 Step 1 — Identifying Entities

**Use the nouns you extracted in Part 2. Decide which ones are true entities:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 ENTITY IDENTIFICATION                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THE TEST: Is it an entity?                                     │
│  ─────────────────────────                                      │
│  Ask three questions:                                           │
│                                                                 │
│  1. Does it have its own IDENTITY? (Can I give it an ID?)       │
│  2. Does it have a LIFECYCLE? (Created, modified, deleted?)     │
│  3. Would I need to REFER to it from other parts of the system? │
│                                                                 │
│  If yes to all three → it's an ENTITY                           │
│  If no → it's probably an ATTRIBUTE of another entity           │
│           or a VALUE with no independent existence               │
│                                                                 │
│                                                                 │
│  APPLYING THE TEST:                                             │
│  ──────────────────                                             │
│                                                                 │
│  Noun              │ Identity? │ Lifecycle? │ Referred? │ Verdict│
│  ──────────────────┼───────────┼────────────┼───────────┼────────│
│  User              │ ✅ user_id│ ✅ reg/del │ ✅ assign │ ENTITY │
│  Organization      │ ✅ org_id │ ✅ crt/del │ ✅ tenant │ ENTITY │
│  Project           │ ✅ proj_id│ ✅ crt/del │ ✅ tasks  │ ENTITY │
│  Task              │ ✅ task_id│ ✅ crt/del │ ✅ assign │ ENTITY │
│  Comment           │ ✅ cmt_id │ ✅ crt/del │ ✅ reply  │ ENTITY │
│  Membership        │ ✅ mbr_id │ ✅ invite  │ ✅ roles  │ ENTITY │
│  AuditLog          │ ✅ log_id │ ✅ create  │ ✅ review │ ENTITY │
│  Notification      │ ✅ ntf_id │ ✅ crt/read│ ✅ user   │ ENTITY │
│  Attachment        │ ✅ att_id │ ✅ upload  │ ✅ tasks  │ ENTITY │
│  ──────────────────┼───────────┼────────────┼───────────┼────────│
│  "status"          │ ❌        │ ❌         │ ❌        │ ATTR   │
│  "priority"        │ ❌        │ ❌         │ ❌        │ ATTR   │
│  "due date"        │ ❌        │ ❌         │ ❌        │ ATTR   │
│  "role"            │ ❌        │ ❌         │ ❌        │ ATTR   │
│  "email"           │ ❌        │ ❌         │ ❌        │ ATTR   │
│  ──────────────────┴───────────┴────────────┴───────────┴────────│
│                                                                 │
│  Result: 9 entities.                                            │
│  This is your system. These are the THINGS you're managing.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice: 'Membership' is its own entity, not an attribute of User or Organization. Why? Because a membership has its own properties (role, joined date) and its own lifecycle (invite → active → revoked). This is the junction table concept from Week 5, but arrived at through domain thinking, not database thinking."

---

## 3.3 Step 2 — Defining Attributes

**For each entity, list its key attributes. Don't aim for completeness — aim for correctness.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  ENTITY ATTRIBUTES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User                          Organization                     │
│  ────                          ────────────                     │
│  id: UUID                      id: UUID                         │
│  email: str (unique)           name: str                        │
│  hashed_password: str          slug: str (unique, URL-safe)     │
│  display_name: str             created_at: datetime             │
│  is_active: bool               updated_at: datetime             │
│  created_at: datetime                                           │
│  updated_at: datetime                                           │
│                                                                 │
│                                                                 │
│  Membership                    Project                          │
│  ──────────                    ───────                          │
│  id: UUID                      id: UUID                         │
│  user_id: FK → User            name: str                        │
│  org_id: FK → Organization     description: str (optional)      │
│  role: enum(admin/member/      org_id: FK → Organization        │
│        viewer)                 created_by: FK → User             │
│  joined_at: datetime           created_at: datetime              │
│                                updated_at: datetime              │
│                                                                 │
│                                                                 │
│  Task                          Comment                          │
│  ────                          ───────                          │
│  id: UUID                      id: UUID                         │
│  title: str                    content: str                     │
│  description: str (optional)   task_id: FK → Task               │
│  status: enum(todo/in_prog/    author_id: FK → User             │
│          review/done)          created_at: datetime              │
│  priority: enum(low/med/       updated_at: datetime             │
│            high/urgent)                                         │
│  assignee_id: FK → User (opt)                                   │
│  project_id: FK → Project                                       │
│  due_date: date (optional)                                      │
│  created_by: FK → User                                          │
│  created_at: datetime                                           │
│  updated_at: datetime                                           │
│                                                                 │
│                                                                 │
│  AuditLog                      Notification                     │
│  ────────                      ────────────                     │
│  id: UUID                      id: UUID                         │
│  org_id: FK → Organization     user_id: FK → User               │
│  actor_id: FK → User           org_id: FK → Organization        │
│  action: str                   type: str                        │
│  entity_type: str              title: str                       │
│  entity_id: UUID               body: str (optional)             │
│  changes: JSON (optional)      is_read: bool                    │
│  timestamp: datetime           related_entity_type: str (opt)   │
│                                related_entity_id: UUID (opt)    │
│                                created_at: datetime             │
│                                                                 │
│                                                                 │
│  Attachment                                                     │
│  ──────────                                                     │
│  id: UUID                                                       │
│  filename: str                                                  │
│  file_url: str                                                  │
│  file_size: int                                                 │
│  content_type: str                                              │
│  task_id: FK → Task                                             │
│  uploaded_by: FK → User                                         │
│  created_at: datetime                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Notice the patterns:**

```
┌─────────────────────────────────────────────────────────────────┐
│               COMMON ATTRIBUTE PATTERNS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ① EVERY entity has: id, created_at                             │
│     Most have: updated_at                                       │
│     → These become a base mixin in your SQLAlchemy models       │
│       (Remember mixins from Week 6? This is why they exist.)    │
│                                                                 │
│  ② Multi-tenant entities have: org_id                           │
│     Task doesn't have org_id directly — it reaches org          │
│     through Project. But AuditLog, Notification do.             │
│     → This matters for query efficiency and data isolation.     │
│                                                                 │
│  ③ Entities with authors have: created_by or author_id          │
│     → Tracks WHO created something (audit trail support).       │
│                                                                 │
│  ④ Enums instead of raw strings for constrained values          │
│     status, priority, role → Python enums + Pydantic            │
│     validation. NOT free-form strings.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Step 3 — Mapping Relationships

**How do these entities connect?**

```
┌─────────────────────────────────────────────────────────────────┐
│                  RELATIONSHIP MAP                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│         ┌──────────┐    belongs to     ┌──────────────┐         │
│         │          │   (many-to-many   │              │         │
│         │   User   │   via Membership) │ Organization │         │
│         │          │◀─────────────────▶│              │         │
│         └────┬─────┘                   └──────┬───────┘         │
│              │                                │                 │
│              │ created_by                     │ has many        │
│              │ assignee                       │                 │
│              │                         ┌──────▼───────┐         │
│              │                         │              │         │
│              │                         │   Project    │         │
│              │                         │              │         │
│              │                         └──────┬───────┘         │
│              │                                │                 │
│              │          assigned_to           │ has many        │
│              │◀──────────────────┐            │                 │
│              │                   │     ┌──────▼───────┐         │
│              │                   │     │              │         │
│              │                   ├─────│    Task      │         │
│              │   author          │     │              │         │
│              │◀──────────┐       │     └──┬───────┬───┘         │
│              │           │       │        │       │             │
│              │     ┌─────▼──┐    │   ┌────▼──┐ ┌──▼────────┐   │
│              │     │Comment │    │   │Attach-│ │Audit Log  │   │
│              │     │        │    │   │ment   │ │(system-   │   │
│              │     └────────┘    │   └───────┘ │ wide)     │   │
│              │                   │              └───────────┘   │
│              │                   │                              │
│         ┌────▼──────────┐   ┌────▼──────────┐                   │
│         │ Notification  │   │  Membership   │                   │
│         │ (to user)     │   │  (user+org    │                   │
│         │               │   │   +role)      │                   │
│         └───────────────┘   └───────────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Formalize the relationships:**

```
┌─────────────────────────────────────────────────────────────────┐
│                RELATIONSHIP TABLE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Relationship                      │  Type        │ FK Location │
│  ──────────────────────────────────┼──────────────┼─────────────│
│  User ↔ Organization               │ Many-to-Many │ Membership  │
│  Organization → Project             │ One-to-Many  │ Project     │
│  Project → Task                     │ One-to-Many  │ Task        │
│  Task → Comment                     │ One-to-Many  │ Comment     │
│  Task → Attachment                  │ One-to-Many  │ Attachment  │
│  User → Task (assigned)             │ One-to-Many  │ Task        │
│  User → Comment (authored)          │ One-to-Many  │ Comment     │
│  User → Notification                │ One-to-Many  │ Notification│
│  Organization → AuditLog            │ One-to-Many  │ AuditLog    │
│  User → AuditLog (actor)            │ One-to-Many  │ AuditLog    │
│  ──────────────────────────────────┴──────────────┴─────────────│
│                                                                 │
│  KEY INSIGHT: User is the most CONNECTED entity.                │
│  It touches almost everything. This is typical — the "actor"    │
│  in a system is always heavily referenced.                      │
│                                                                 │
│  KEY INSIGHT: The data access PATH matters.                     │
│  To get a Task, you traverse: Org → Project → Task              │
│  This path IS your URL structure: /orgs/{id}/projects/{id}/...  │
│  The domain model is already shaping your API.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Step 4 — Grouping into Modules (Boundaries)

**Not all entities belong together. Some are closer than others.**

> "In a house, the bathroom fixtures are grouped together because they share plumbing. You don't run a shower pipe through the kitchen. In your code, entities that change together and are used together should live together."

```
┌─────────────────────────────────────────────────────────────────┐
│                   MODULE BOUNDARIES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Ask: "Which entities are ALWAYS used together?"                │
│       "If I change this entity, what else MUST change?"         │
│       "Can I develop/test this group independently?"            │
│                                                                 │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  AUTH MODULE                                            │    │
│  │  ─────────────                                          │    │
│  │  Entities: User (auth-related attributes only)          │    │
│  │  Concerns: Registration, login, tokens, password reset  │    │
│  │  Changes independently from projects/tasks              │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  ORGANIZATION MODULE                                    │    │
│  │  ───────────────────                                    │    │
│  │  Entities: Organization, Membership                     │    │
│  │  Concerns: Org CRUD, invitations, role management       │    │
│  │  Owns the multi-tenancy boundary                        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  PROJECT MODULE                                         │    │
│  │  ──────────────                                         │    │
│  │  Entities: Project, Task, Comment, Attachment           │    │
│  │  Concerns: The CORE domain — project management itself  │    │
│  │  This is where most business logic lives                │    │
│  │  Task/Comment/Attachment don't make sense without       │    │
│  │  Project — they're a cohesive group                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  NOTIFICATION MODULE                                    │    │
│  │  ───────────────────                                    │    │
│  │  Entities: Notification                                 │    │
│  │  Concerns: Creating, delivering, marking as read        │    │
│  │  WebSocket connection management                        │    │
│  │  Reacts to events from OTHER modules (not originator)   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  AUDIT MODULE                                           │    │
│  │  ────────────                                           │    │
│  │  Entities: AuditLog                                     │    │
│  │  Concerns: Recording all mutations across the system    │    │
│  │  Read-heavy, append-only, never modified                │    │
│  │  Also reacts to events — never originates them          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│                                                                 │
│  WHY THESE GROUPINGS?                                           │
│  ────────────────────                                           │
│  • Auth changes (e.g. add OAuth) should NOT affect tasks        │
│  • Adding a new project feature should NOT touch auth code      │
│  • Notification logic can be developed/tested independently     │
│  • Audit logging can be swapped (DB → external service)         │
│    without changing anything else                               │
│                                                                 │
│  Each module has its own: models, schemas, router, service,     │
│  repository. This becomes your FOLDER STRUCTURE.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "Why did I put Comment inside the Project module and not in its own module?"

Answer: A comment has no meaning outside of a task, which has no meaning outside of a project. They share a lifecycle boundary. You would never build a 'comment service' that operates independently — it always needs the task context. Entities that have no independent existence belong in the module of their parent.

> "Why is Notification a separate module from Project, even though notifications are TRIGGERED by project events?"

Answer: The notification module doesn't know *why* it's sending a notification. It just receives an instruction: "Tell user X that Y happened." The project module doesn't know *how* notifications are delivered (WebSocket? Email? Push?). This decoupling means you can change notification delivery without touching project logic. You've seen this pattern before — the event-driven concepts from Week 11.

---

## 3.6 From Domain Model to ER Diagram

**Now — and only now — translate to a database schema.**

**Connection to Week 5:**

> "In Week 5, you learned to draw ER diagrams from requirements directly. That works for small systems. For larger systems, you pass through the domain model first. The domain model is the thinking; the ER diagram is the result."

```
┌─────────────────────────────────────────────────────────────────┐
│               DOMAIN MODEL → ER DIAGRAM                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What changes when you translate:                               │
│                                                                 │
│  DOMAIN CONCEPT              DATABASE IMPLEMENTATION            │
│  ──────────────────────────  ──────────────────────────         │
│  Entity "User"               Table "users"                      │
│  Attribute "email: str"      Column email VARCHAR(255) UNIQUE   │
│  "enum(admin/member/viewer)" Column role VARCHAR(20) + CHECK    │
│                              OR separate enum type               │
│  "created_at: datetime"      Column TIMESTAMPTZ DEFAULT now()   │
│  "id: UUID"                  Column UUID DEFAULT gen_random_uuid│
│  Many-to-many (User↔Org)    Junction table "memberships"       │
│  "changes: JSON"             Column JSONB (PostgreSQL)          │
│                                                                 │
│                                                                 │
│  DECISIONS YOU MAKE DURING TRANSLATION:                         │
│  ──────────────────────────────────────                         │
│  1. UUID vs serial for PKs (you discussed this in Week 5)       │
│     → UUID: better for distributed systems, harder to guess     │
│     → Serial: simpler, better index performance                 │
│     → For multi-tenant SaaS: UUID is the safer choice           │
│                                                                 │
│  2. Soft delete vs hard delete (also Week 5)                    │
│     → AuditLog: never deleted (append-only)                     │
│     → User: soft delete (is_active = false)                     │
│     → Comment: hard delete (or soft, depending on policy)       │
│                                                                 │
│  3. Indexing strategy (Week 7)                                  │
│     → Every FK gets an index (you'll query by org_id, etc.)     │
│     → Composite index on (org_id, created_at) for audit logs    │
│     → Partial index on tasks WHERE status != 'done'             │
│                                                                 │
│  These decisions belong in an ADR (Part 5).                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "See the progression? Business email → nouns/verbs → entities → attributes → relationships → modules → ER diagram. At no point did we open an editor. The THINKING is the work. The typing is just transcription."

---

# PART 4: PROJECT STRUCTURE

## 4.1 The Layered Architecture

**Your domain model gives you the modules. Now you need the LAYERS within each module.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE FOUR LAYERS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LAYER 1: ROUTER (the door)                                     │
│  ─────────────────────────                                      │
│  Responsibility: HTTP concerns ONLY.                            │
│  ├─ Defines endpoints (path, method, status codes)              │
│  ├─ Receives request → validates via Pydantic (schemas)         │
│  ├─ Calls the service layer                                     │
│  ├─ Returns response                                            │
│  └─ Knows: HTTP, Pydantic schemas, FastAPI                      │
│     Does NOT know: SQL, database tables, Redis, Celery          │
│                                                                 │
│  LAYER 2: SERVICE (the brain)                                   │
│  ────────────────────────────                                   │
│  Responsibility: BUSINESS LOGIC.                                │
│  ├─ Orchestrates operations (validate, transform, coordinate)   │
│  ├─ Enforces business rules ("admins only", "max 50 projects")  │
│  ├─ Calls repositories for data access                          │
│  ├─ Calls external services (notifications, background jobs)    │
│  └─ Knows: domain rules, repository interfaces                  │
│     Does NOT know: HTTP, status codes, request/response format  │
│                                                                 │
│  LAYER 3: REPOSITORY (the hands)                                │
│  ────────────────────────────────                               │
│  Responsibility: DATA ACCESS ONLY.                              │
│  ├─ Executes database queries (SQLAlchemy)                      │
│  ├─ Translates between DB rows and domain objects               │
│  ├─ Handles pagination, filtering at the query level            │
│  └─ Knows: SQLAlchemy, SQL, database models                     │
│     Does NOT know: HTTP, business rules, Pydantic schemas       │
│                                                                 │
│  LAYER 4: MODEL (the skeleton)                                  │
│  ────────────────────────────                                   │
│  Responsibility: DATA DEFINITION.                               │
│  ├─ SQLAlchemy model classes (table definitions)                │
│  ├─ Pydantic schema classes (request/response shapes)           │
│  └─ Pure data — no logic, no side effects                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to Week 6:**

> "You introduced the repository pattern in Week 6 when you separated database queries from your route handlers. That was the seed. Now we add the service layer between the router and repository. The pattern you already know is one layer of a larger architecture."

---

## 4.2 Request Flow Through Layers

**Trace a single request from HTTP to database and back:**

```
┌─────────────────────────────────────────────────────────────────┐
│          REQUEST: POST /orgs/{org_id}/projects                  │
│          "Create a new project in an organization"              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT                                                         │
│    │                                                            │
│    │  POST /orgs/abc-123/projects                               │
│    │  Authorization: Bearer eyJ...                              │
│    │  Body: {"name": "Website Redesign"}                        │
│    │                                                            │
│    ▼                                                            │
│  ┌──────────────────────────────────────────────────────┐       │
│  │  ROUTER  (projects/router.py)                        │       │
│  │                                                      │       │
│  │  @router.post("/orgs/{org_id}/projects",             │       │
│  │               status_code=201)                       │       │
│  │  async def create_project(                           │       │
│  │      org_id: UUID,                                   │       │
│  │      data: ProjectCreate,     ← Pydantic validates   │       │
│  │      user: User = Depends(get_current_user),         │       │
│  │      service: ProjectService = Depends(),            │       │
│  │  ) -> ProjectResponse:                               │       │
│  │      project = await service.create_project(         │       │
│  │          org_id=org_id,                              │       │
│  │          data=data,                                  │       │
│  │          user=user,                                  │       │
│  │      )                                               │       │
│  │      return project     ← Pydantic serializes        │       │
│  │                                                      │       │
│  │  The router does NOT check permissions.              │       │
│  │  The router does NOT query the database.             │       │
│  │  It DELEGATES to the service.                        │       │
│  └───────────────────────┬──────────────────────────────┘       │
│                          │                                      │
│                          ▼                                      │
│  ┌──────────────────────────────────────────────────────┐       │
│  │  SERVICE  (projects/service.py)                      │       │
│  │                                                      │       │
│  │  async def create_project(self, org_id, data, user): │       │
│  │      # 1. Check permission: is user a member/admin?  │       │
│  │      membership = await self.org_repo                │       │
│  │          .get_membership(user.id, org_id)            │       │
│  │      if not membership or membership.role == "viewer"│       │
│  │          raise PermissionDeniedError()               │       │
│  │                                                      │       │
│  │      # 2. Check business rule: max 50 projects/org   │       │
│  │      count = await self.project_repo                 │       │
│  │          .count_by_org(org_id)                       │       │
│  │      if count >= 50:                                 │       │
│  │          raise ProjectLimitExceededError()            │       │
│  │                                                      │       │
│  │      # 3. Create the project                         │       │
│  │      project = await self.project_repo               │       │
│  │          .create(org_id=org_id, name=data.name,      │       │
│  │                  created_by=user.id)                  │       │
│  │                                                      │       │
│  │      # 4. Side effects                               │       │
│  │      await self.audit_service                        │       │
│  │          .log("create", "project", project.id, user) │       │
│  │                                                      │       │
│  │      return project                                  │       │
│  │                                                      │       │
│  │  The service does NOT know about HTTP status codes.  │       │
│  │  It raises DOMAIN exceptions, not HTTPException.     │       │
│  │  It does NOT write SQL.                              │       │
│  └───────────────────────┬──────────────────────────────┘       │
│                          │                                      │
│                          ▼                                      │
│  ┌──────────────────────────────────────────────────────┐       │
│  │  REPOSITORY  (projects/repository.py)                │       │
│  │                                                      │       │
│  │  async def create(self, org_id, name, created_by):   │       │
│  │      project = Project(                              │       │
│  │          org_id=org_id,                              │       │
│  │          name=name,                                  │       │
│  │          created_by=created_by,                      │       │
│  │      )                                               │       │
│  │      self.session.add(project)                       │       │
│  │      await self.session.flush()                      │       │
│  │      return project                                  │       │
│  │                                                      │       │
│  │  The repository does NOT check permissions.          │       │
│  │  The repository does NOT enforce business limits.    │       │
│  │  It ONLY translates between Python objects and SQL.  │       │
│  └───────────────────────┬──────────────────────────────┘       │
│                          │                                      │
│                          ▼                                      │
│                    ┌────────────┐                                │
│                    │ PostgreSQL │                                │
│                    └────────────┘                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "Where do domain exceptions become HTTP status codes?"

Answer: **At the boundary between router and service.** The router catches `PermissionDeniedError` and returns 403. The router catches `ProjectLimitExceededError` and returns 422 or 409. The service never knows about HTTP. This is separation of concerns.

```python
# In the router or via a global exception handler (Week 3 concept):

@app.exception_handler(PermissionDeniedError)
async def handle_permission_denied(request, exc):
    return JSONResponse(status_code=403, content={"detail": str(exc)})

@app.exception_handler(ProjectLimitExceededError)
async def handle_limit_exceeded(request, exc):
    return JSONResponse(status_code=409, content={"detail": str(exc)})
```

> "You learned custom exception handlers in Week 3. THIS is where they become critical. Domain exceptions are translated to HTTP responses at the edge."

---

## 4.3 The Dependency Rule

**The single most important rule in layered architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE DEPENDENCY RULE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Dependencies point INWARD. Never outward.                      │
│                                                                 │
│                                                                 │
│      OUTER LAYERS (know about the world)                        │
│      ┌─────────────────────────────────────────────┐            │
│      │  Router                                     │            │
│      │  (knows HTTP, knows service exists)          │            │
│      │  ┌─────────────────────────────────────┐    │            │
│      │  │  Service                            │    │            │
│      │  │  (knows business rules,             │    │            │
│      │  │   knows repository exists)          │    │            │
│      │  │  ┌─────────────────────────────┐    │    │            │
│      │  │  │  Repository                 │    │    │            │
│      │  │  │  (knows database,           │    │    │            │
│      │  │  │   knows models exist)       │    │    │            │
│      │  │  │  ┌─────────────────────┐    │    │    │            │
│      │  │  │  │  Models / Schemas   │    │    │    │            │
│      │  │  │  │  (knows nothing)    │    │    │    │            │
│      │  │  │  └─────────────────────┘    │    │    │            │
│      │  │  └─────────────────────────────┘    │    │            │
│      │  └─────────────────────────────────────┘    │            │
│      └─────────────────────────────────────────────┘            │
│      INNER LAYERS (pure, know nothing about outer world)        │
│                                                                 │
│                                                                 │
│  ✅ ALLOWED:                    ❌ FORBIDDEN:                    │
│  Router → imports Service       Service → imports Router        │
│  Service → imports Repository   Repository → imports Service    │
│  Repository → imports Model     Model → imports Repository      │
│  Router → imports Schema        Repository → imports Router     │
│                                                                 │
│                                                                 │
│  WHY? Because inner layers are STABLE. They change rarely.      │
│  Outer layers are VOLATILE. They change often.                  │
│  If Repository imported Router, changing a URL would break      │
│  your database code. That's absurd. Dependencies flow           │
│  toward stability.                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A practical test:**

> "Can I delete my entire router layer and replace it with a CLI interface, without changing a single line in my services or repositories? If yes, your dependency direction is correct. If no, your layers are leaking."

---

## 4.4 The Full Folder Layout

**Now we translate modules × layers into a concrete folder structure.**

```
┌─────────────────────────────────────────────────────────────────┐
│                PROJECT FOLDER STRUCTURE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  project-root/                                                  │
│  ├── app/                        # Application package          │
│  │   ├── __init__.py                                            │
│  │   ├── main.py                 # FastAPI app factory          │
│  │   │                                                          │
│  │   ├── core/                   # Shared infrastructure        │
│  │   │   ├── __init__.py                                        │
│  │   │   ├── config.py           # pydantic-settings (Week 15)  │
│  │   │   ├── database.py         # engine, session factory      │
│  │   │   ├── dependencies.py     # shared Depends (get_db, etc) │
│  │   │   ├── security.py         # JWT, password hashing        │
│  │   │   ├── exceptions.py       # domain exception hierarchy   │
│  │   │   └── redis.py            # Redis client factory         │
│  │   │                                                          │
│  │   ├── auth/                   # Auth module                  │
│  │   │   ├── __init__.py                                        │
│  │   │   ├── router.py           # /auth/register, /auth/login  │
│  │   │   ├── service.py          # registration, verification   │
│  │   │   ├── repository.py       # user queries                 │
│  │   │   ├── schemas.py          # UserCreate, UserResponse...  │
│  │   │   └── dependencies.py     # get_current_user             │
│  │   │                                                          │
│  │   ├── organizations/          # Organization module          │
│  │   │   ├── __init__.py                                        │
│  │   │   ├── router.py           # /orgs CRUD, /orgs/{}/members │
│  │   │   ├── service.py          # org logic, membership mgmt   │
│  │   │   ├── repository.py       # org + membership queries     │
│  │   │   └── schemas.py          # OrgCreate, MemberResponse... │
│  │   │                                                          │
│  │   ├── projects/               # Project module               │
│  │   │   ├── __init__.py                                        │
│  │   │   ├── router.py           # /orgs/{}/projects CRUD       │
│  │   │   ├── service.py          # project business logic       │
│  │   │   ├── repository.py       # project queries              │
│  │   │   └── schemas.py          # ProjectCreate, TaskCreate... │
│  │   │                                                          │
│  │   ├── tasks/                  # Task module (inside project  │
│  │   │   ├── __init__.py         #  domain, but own files for   │
│  │   │   ├── router.py           #  manageability)              │
│  │   │   ├── service.py                                         │
│  │   │   ├── repository.py                                      │
│  │   │   └── schemas.py                                         │
│  │   │                                                          │
│  │   ├── notifications/          # Notification module          │
│  │   │   ├── __init__.py                                        │
│  │   │   ├── router.py           # /notifications, WebSocket    │
│  │   │   ├── service.py          # notification logic           │
│  │   │   ├── repository.py       # notification queries         │
│  │   │   ├── schemas.py                                         │
│  │   │   └── websocket.py        # connection manager           │
│  │   │                                                          │
│  │   ├── audit/                  # Audit module                 │
│  │   │   ├── __init__.py                                        │
│  │   │   ├── router.py           # /orgs/{}/audit-logs (read)   │
│  │   │   ├── service.py          # logging mutations            │
│  │   │   ├── repository.py       # audit log queries            │
│  │   │   └── schemas.py                                         │
│  │   │                                                          │
│  │   └── models/                 # ALL SQLAlchemy models        │
│  │       ├── __init__.py         # (centralized for Alembic)    │
│  │       ├── base.py             # DeclarativeBase, mixins      │
│  │       ├── user.py                                            │
│  │       ├── organization.py                                    │
│  │       ├── project.py                                         │
│  │       ├── task.py                                            │
│  │       ├── comment.py                                         │
│  │       ├── notification.py                                    │
│  │       ├── audit_log.py                                       │
│  │       └── attachment.py                                      │
│  │                                                              │
│  ├── alembic/                    # Migrations                   │
│  │   ├── versions/                                              │
│  │   └── env.py                                                 │
│  │                                                              │
│  ├── workers/                    # Celery tasks                 │
│  │   ├── __init__.py                                            │
│  │   ├── celery_app.py           # Celery configuration         │
│  │   ├── email_tasks.py          # Send email notifications     │
│  │   └── export_tasks.py         # CSV/report generation        │
│  │                                                              │
│  ├── tests/                      # Mirror the app structure     │
│  │   ├── conftest.py             # Shared fixtures              │
│  │   ├── factories.py            # Test data factories          │
│  │   ├── auth/                                                  │
│  │   │   ├── test_router.py                                     │
│  │   │   ├── test_service.py                                    │
│  │   │   └── test_repository.py                                 │
│  │   ├── organizations/                                         │
│  │   │   └── ...                                                │
│  │   ├── projects/                                              │
│  │   │   └── ...                                                │
│  │   └── tasks/                                                 │
│  │       └── ...                                                │
│  │                                                              │
│  ├── docs/                       # Documentation                │
│  │   └── adr/                    # Architecture Decision Records│
│  │       ├── 001-use-uuid-pks.md                                │
│  │       ├── 002-row-level-multitenancy.md                      │
│  │       └── ...                                                │
│  │                                                              │
│  ├── alembic.ini                                                │
│  ├── pyproject.toml                                             │
│  ├── docker-compose.yml                                         │
│  └── README.md                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why models/ is centralized instead of per-module:**

> "You might ask: why not put User model inside `auth/models.py` and Organization model inside `organizations/models.py`? It's because Alembic needs to discover ALL models to generate migrations. Cross-module relationships (User FK in Task) also create circular import headaches when models are scattered. Centralizing models in one package avoids both problems. The schemas stay per-module because they have no cross-module import issues."

---

## 4.5 Module Isolation — One Domain Per Package

**Each module follows the same internal pattern:**

```
┌─────────────────────────────────────────────────────────────────┐
│              MODULE INTERNAL PATTERN                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  organizations/                                                 │
│  ├── router.py        WHAT:  HTTP endpoint definitions          │
│  │                    IMPORTS: schemas, service, auth deps       │
│  │                                                              │
│  ├── service.py       WHAT:  Business logic, orchestration      │
│  │                    IMPORTS: repository, domain exceptions     │
│  │                    INJECTED: repository via __init__          │
│  │                                                              │
│  ├── repository.py    WHAT:  Database queries                   │
│  │                    IMPORTS: SQLAlchemy models                 │
│  │                    INJECTED: AsyncSession via __init__        │
│  │                                                              │
│  ├── schemas.py       WHAT:  Pydantic request/response models   │
│  │                    IMPORTS: nothing from this project         │
│  │                                                              │
│  └── dependencies.py  WHAT:  FastAPI Depends for this module    │
│                       IMPORTS: service, repository              │
│                                                                 │
│                                                                 │
│  EVERY module looks like this. Once you understand one,         │
│  you understand them all. The consistency IS the architecture.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How the service gets its dependencies — Dependency Injection with FastAPI:**

**Connection to Week 3:**

> "You learned `Depends()` in Week 3 for injecting database sessions and the current user. Now we scale that same mechanism to wire entire layers together."

```python
# organizations/dependencies.py

from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.dependencies import get_db
from app.organizations.repository import OrganizationRepository
from app.organizations.service import OrganizationService


def get_org_repository(
    session: AsyncSession = Depends(get_db),
) -> OrganizationRepository:
    return OrganizationRepository(session)


def get_org_service(
    repo: OrganizationRepository = Depends(get_org_repository),
) -> OrganizationService:
    return OrganizationService(repo)
```

```python
# organizations/router.py

from fastapi import APIRouter, Depends, status
from app.auth.dependencies import get_current_user
from app.organizations.dependencies import get_org_service
from app.organizations.service import OrganizationService
from app.organizations.schemas import OrgCreate, OrgResponse
from app.models.user import User

router = APIRouter(prefix="/orgs", tags=["organizations"])


@router.post("/", status_code=status.HTTP_201_CREATED, response_model=OrgResponse)
async def create_organization(
    data: OrgCreate,
    user: User = Depends(get_current_user),
    service: OrganizationService = Depends(get_org_service),
) -> OrgResponse:
    return await service.create_organization(data=data, user=user)
```

```python
# organizations/service.py

from app.organizations.repository import OrganizationRepository
from app.organizations.schemas import OrgCreate
from app.core.exceptions import DuplicateSlugError
from app.models.user import User


class OrganizationService:
    def __init__(self, repo: OrganizationRepository) -> None:
        self.repo = repo

    async def create_organization(self, data: OrgCreate, user: User):
        # Business rule: slug must be unique
        existing = await self.repo.get_by_slug(data.slug)
        if existing:
            raise DuplicateSlugError(data.slug)

        org = await self.repo.create(name=data.name, slug=data.slug)

        # Creator automatically becomes admin
        await self.repo.add_member(
            org_id=org.id, user_id=user.id, role="admin"
        )

        return org
```

```python
# organizations/repository.py

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.organization import Organization
from app.models.membership import Membership


class OrganizationRepository:
    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    async def create(self, name: str, slug: str) -> Organization:
        org = Organization(name=name, slug=slug)
        self.session.add(org)
        await self.session.flush()
        return org

    async def get_by_slug(self, slug: str) -> Organization | None:
        result = await self.session.execute(
            select(Organization).where(Organization.slug == slug)
        )
        return result.scalar_one_or_none()

    async def add_member(
        self, org_id, user_id, role: str
    ) -> Membership:
        membership = Membership(
            org_id=org_id, user_id=user_id, role=role
        )
        self.session.add(membership)
        await self.session.flush()
        return membership
```

**Now look at what we've achieved:**

```
┌─────────────────────────────────────────────────────────────────┐
│               WHAT LAYERING GIVES YOU                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TESTING:                                                       │
│  ├─ Test the service with a FAKE repository (no database)       │
│  ├─ Test the repository against a real test database            │
│  ├─ Test the router with a mocked service                       │
│  └─ Each layer tested INDEPENDENTLY                             │
│     (Remember dependency_overrides from Week 4? This is why.)   │
│                                                                 │
│  READABILITY:                                                   │
│  ├─ "Where's the org creation logic?" → service.py              │
│  ├─ "Where's the org query?" → repository.py                    │
│  ├─ "What's the request format?" → schemas.py                   │
│  └─ Every file has ONE job. You never search through 3000 lines.│
│                                                                 │
│  CHANGEABILITY:                                                 │
│  ├─ Switch from PostgreSQL to MongoDB? Change repositories only.│
│  ├─ Add GraphQL alongside REST? Add new routers, same services. │
│  ├─ Change a business rule? Edit one service method.            │
│  └─ Each change is LOCALIZED.                                   │
│                                                                 │
│  COLLABORATION:                                                 │
│  ├─ Dev A works on organizations/service.py                     │
│  ├─ Dev B works on tasks/service.py                             │
│  └─ No merge conflicts. Different files, different modules.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## How main.py Ties It All Together

**The app factory — clean and minimal:**

```python
# app/main.py

from fastapi import FastAPI

from app.auth.router import router as auth_router
from app.organizations.router import router as org_router
from app.projects.router import router as project_router
from app.tasks.router import router as task_router
from app.notifications.router import router as notification_router
from app.audit.router import router as audit_router
from app.core.exceptions import register_exception_handlers


def create_app() -> FastAPI:
    app = FastAPI(
        title="Project Management SaaS",
        version="1.0.0",
    )

    # Register exception handlers (domain errors → HTTP responses)
    register_exception_handlers(app)

    # Mount module routers
    app.include_router(auth_router)
    app.include_router(org_router)
    app.include_router(project_router)
    app.include_router(task_router)
    app.include_router(notification_router)
    app.include_router(audit_router)

    return app


app = create_app()
```

> "Compare this to the 200+ line `main.py` from Part 1. This file is ~30 lines. It does ONE thing: assembles the application from modules. If you need to understand the task system, you open `tasks/`. You never touch this file."

---

# PART 5: ARCHITECTURE DECISION RECORDS

## 5.1 Why Document Decisions?

```
┌─────────────────────────────────────────────────────────────────┐
│             WHY WRITE ADRs?                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Every architecture has CHOICES. You chose:                     │
│  • UUID over serial IDs                                         │
│  • Row-level multi-tenancy over schema-per-tenant               │
│  • Celery over FastAPI BackgroundTasks for heavy jobs            │
│  • Centralized models/ over per-module models                   │
│                                                                 │
│  Six months from now, you (or a new teammate) will ask:         │
│                                                                 │
│      "Why did we use UUIDs? Serial would be simpler..."         │
│                                                                 │
│  Without an ADR:                                                │
│  ├─ Nobody remembers why                                        │
│  ├─ Someone "fixes" it by switching to serial                   │
│  ├─ Three services break because they assumed UUID format       │
│  └─ Two days wasted rediscovering the original reasoning        │
│                                                                 │
│  With an ADR:                                                   │
│  ├─ Open docs/adr/001-use-uuid-pks.md                          │
│  ├─ Read the context and tradeoffs                              │
│  ├─ Understand why the decision was made                        │
│  ├─ Make an INFORMED choice about whether to change it          │
│  └─ Five minutes instead of two days                            │
│                                                                 │
│                                                                 │
│  An ADR is not bureaucracy. It's a LETTER TO YOUR FUTURE SELF.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 The ADR Template

```
┌─────────────────────────────────────────────────────────────────┐
│                   ADR TEMPLATE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  # ADR-{NUMBER}: {TITLE}                                        │
│                                                                 │
│  ## Status                                                      │
│  {Proposed | Accepted | Deprecated | Superseded by ADR-XXX}     │
│                                                                 │
│  ## Context                                                     │
│  What is the situation? What forces are at play?                │
│  What problem are we trying to solve?                           │
│  (Keep it factual. Describe the SITUATION, not your opinion.)   │
│                                                                 │
│  ## Decision                                                    │
│  What did we decide? State it clearly in one sentence.          │
│  Then explain the reasoning.                                    │
│                                                                 │
│  ## Consequences                                                │
│  What are the results of this decision?                         │
│  List BOTH positive and negative consequences.                  │
│  Be honest about the tradeoffs.                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Two Real ADRs for the Capstone

**ADR #1:**

```markdown
# ADR-001: Use UUIDs for Primary Keys

## Status
Accepted

## Context
We need to choose a primary key strategy for all database tables.
The two main options are auto-incrementing serial integers and UUIDs.

This is a multi-tenant SaaS application where:
- Data may be exposed in API URLs (/orgs/{id}/tasks/{id})
- Multiple services or clients might create records
- Sequential IDs leak information (competitor could estimate
  our total number of organizations by creating one and seeing
  the ID)

## Decision
We will use UUID v4 for all primary keys, generated at the
database level using PostgreSQL's gen_random_uuid().

Reasoning:
- IDs in URLs don't reveal record count or creation order
- No coordination needed for ID generation across services
- Safe to include in logs and error messages

## Consequences
POSITIVE:
- No information leakage through sequential IDs
- IDs can be generated client-side if needed in future
- Globally unique — safe across tables and services

NEGATIVE:
- Larger storage (16 bytes vs 4 bytes for int)
- Slightly slower index lookups vs sequential integers
- Harder to read/type in debugging ("which task was abc-123...?")
- Random UUIDs cause B-tree index fragmentation
  (mitigated by UUID v7 if we upgrade later)

We accept these tradeoffs given the SaaS context.
```

**ADR #2:**

```markdown
# ADR-002: Row-Level Multi-Tenancy

## Status
Accepted

## Context
Organizations must be isolated — Org A must never see Org B's data.
Multi-tenancy can be implemented via:

1. Separate databases per tenant (strongest isolation, highest cost)
2. Separate schemas per tenant (good isolation, complex migrations)
3. Row-level isolation with org_id filtering (simplest, shared tables)

We expect ~50 organizations in year one with ~500 total users.
We have one developer. We use Alembic for migrations.

## Decision
We will use row-level multi-tenancy: all tenants share the
same tables, with org_id columns and query-level filtering.

We will enforce isolation through:
- A reusable dependency that extracts org_id from the URL
- Repository methods that ALWAYS filter by org_id
- Integration tests that verify cross-tenant data is inaccessible

## Consequences
POSITIVE:
- Single database, single schema, single migration path
- Simple Alembic workflow (one versions/ directory)
- Low infrastructure cost at our scale
- Easy to query across tenants for admin/analytics

NEGATIVE:
- ONE missed WHERE clause leaks data between tenants
  (mitigated by repository pattern + tests)
- No database-level isolation (a bad query CAN access all data)
- Harder to scale per-tenant if one org is much larger than others
- Harder to comply with "delete all my data" if required later

If we outgrow this approach, we will migrate to schema-per-tenant.
This decision will be revisited at 500+ organizations.
```

> "Notice: ADRs don't just say WHAT you decided. They document WHY, and what the DOWNSIDES are. Future you needs the context, not just the conclusion."

---

# PART 6: API CONTRACT FIRST

## 6.1 Design Before Implementation

```
┌─────────────────────────────────────────────────────────────────┐
│                 CONTRACT-FIRST DESIGN                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TRADITIONAL APPROACH (code-first):                             │
│  ──────────────────────────────────                             │
│  Write code → API docs generated automatically                  │
│  Problem: you discover design issues WHILE coding               │
│  "Oh wait, this endpoint needs the org_id too..."               │
│  Constant refactoring.                                          │
│                                                                 │
│                                                                 │
│  CONTRACT-FIRST APPROACH:                                       │
│  ────────────────────────                                       │
│  Design all endpoints → Write schemas → THEN implement          │
│  You discover design issues BEFORE touching the database.       │
│  "Wait, should task creation return the full project or         │
│   just the task? Let's decide NOW, not during sprint 3."        │
│                                                                 │
│                                                                 │
│  The contract is:                                               │
│  ├─ Every endpoint path and HTTP method                         │
│  ├─ Every request body shape (Pydantic schema)                  │
│  ├─ Every response body shape (Pydantic schema)                 │
│  ├─ Every error response and status code                        │
│  └─ Auth requirements per endpoint                              │
│                                                                 │
│  If a frontend team existed, they could start building          │
│  against your contract TODAY while you implement tomorrow.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to Week 3:**

> "You learned that FastAPI auto-generates OpenAPI docs from your code. In contract-first design, you flip this: you write the Pydantic schemas and router stubs FIRST, see the generated docs in Swagger UI, verify the API makes sense, and THEN fill in the implementation."

---

## 6.2 Schema-First with Pydantic

**Start with the data shapes. No logic yet.**

```python
# tasks/schemas.py — Written BEFORE any service or repository code

from datetime import date, datetime
from enum import Enum
from uuid import UUID

from pydantic import BaseModel, Field


class TaskStatus(str, Enum):
    TODO = "todo"
    IN_PROGRESS = "in_progress"
    IN_REVIEW = "in_review"
    DONE = "done"


class TaskPriority(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    URGENT = "urgent"


# --- Request Schemas ---

class TaskCreate(BaseModel):
    title: str = Field(min_length=1, max_length=300)
    description: str | None = None
    priority: TaskPriority = TaskPriority.MEDIUM
    assignee_id: UUID | None = None
    due_date: date | None = None


class TaskUpdate(BaseModel):
    """All fields optional — partial update (PATCH semantics)"""
    title: str | None = Field(default=None, min_length=1, max_length=300)
    description: str | None = None
    status: TaskStatus | None = None
    priority: TaskPriority | None = None
    assignee_id: UUID | None = None
    due_date: date | None = None


class TaskFilter(BaseModel):
    """Query parameters for listing tasks"""
    status: TaskStatus | None = None
    priority: TaskPriority | None = None
    assignee_id: UUID | None = None
    search: str | None = None


# --- Response Schemas ---

class TaskSummary(BaseModel):
    """Light version for list endpoints"""
    id: UUID
    title: str
    status: TaskStatus
    priority: TaskPriority
    assignee_id: UUID | None
    due_date: date | None
    created_at: datetime
    model_config = {"from_attributes": True}


class TaskDetail(TaskSummary):
    """Full version for detail endpoint"""
    description: str | None
    project_id: UUID
    created_by: UUID
    updated_at: datetime
    comment_count: int
    attachment_count: int


# --- Pagination (reusable, probably in core/schemas.py) ---

class PaginatedResponse[T](BaseModel):
    items: list[T]
    total: int
    page: int
    page_size: int
    has_next: bool
```

**Notice the design decisions embedded in the schemas:**

```
┌─────────────────────────────────────────────────────────────────┐
│           DESIGN DECISIONS IN SCHEMAS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ① TaskCreate vs TaskUpdate — separate schemas                  │
│     Create: title is REQUIRED                                   │
│     Update: title is OPTIONAL (partial update)                  │
│     You learned this separation in Week 3 (Pydantic lecture).   │
│                                                                 │
│  ② TaskSummary vs TaskDetail — two response shapes              │
│     List endpoint: returns lightweight summaries (fast)         │
│     Detail endpoint: returns everything (one item, worth it)    │
│     This prevents over-fetching on list endpoints.              │
│                                                                 │
│  ③ comment_count, attachment_count in TaskDetail                │
│     Instead of embedding full comments in the task response,    │
│     return a COUNT. Client fetches comments separately.         │
│     Avoids deeply nested responses.                             │
│                                                                 │
│  ④ TaskFilter as a schema for query parameters                  │
│     Groups all filter options into one validated object.        │
│     Reusable, testable, self-documenting.                       │
│                                                                 │
│  ⑤ Enums for status and priority                                │
│     Not strings. Pydantic rejects invalid values at the door.   │
│     Validation as security, as you learned in Week 9.           │
│                                                                 │
│  All of this decided WITHOUT writing a single query.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.3 Router Stubs — The Skeleton

**Write the endpoints as empty shells. Verify the API surface in Swagger before implementing.**

```python
# tasks/router.py — STUB: no implementation yet

from uuid import UUID

from fastapi import APIRouter, Depends, Query, status

from app.auth.dependencies import get_current_user
from app.models.user import User
from app.tasks.schemas import (
    TaskCreate,
    TaskDetail,
    TaskFilter,
    TaskSummary,
    TaskUpdate,
    PaginatedResponse,
)

router = APIRouter(
    prefix="/orgs/{org_id}/projects/{project_id}/tasks",
    tags=["tasks"],
)


@router.post(
    "/",
    status_code=status.HTTP_201_CREATED,
    response_model=TaskDetail,
    summary="Create a task",
    responses={
        403: {"description": "Not a member/admin of this org"},
        404: {"description": "Project not found"},
    },
)
async def create_task(
    org_id: UUID,
    project_id: UUID,
    data: TaskCreate,
    user: User = Depends(get_current_user),
):
    ...  # Implementation comes later


@router.get(
    "/",
    response_model=PaginatedResponse[TaskSummary],
    summary="List tasks in a project",
)
async def list_tasks(
    org_id: UUID,
    project_id: UUID,
    user: User = Depends(get_current_user),
    status_filter: str | None = Query(None, alias="status"),
    priority: str | None = None,
    assignee_id: UUID | None = None,
    search: str | None = None,
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
):
    ...


@router.get(
    "/{task_id}",
    response_model=TaskDetail,
    summary="Get task details",
    responses={404: {"description": "Task not found"}},
)
async def get_task(
    org_id: UUID,
    project_id: UUID,
    task_id: UUID,
    user: User = Depends(get_current_user),
):
    ...


@router.patch(
    "/{task_id}",
    response_model=TaskDetail,
    summary="Update a task",
    responses={
        403: {"description": "Viewers cannot modify tasks"},
        404: {"description": "Task not found"},
    },
)
async def update_task(
    org_id: UUID,
    project_id: UUID,
    task_id: UUID,
    data: TaskUpdate,
    user: User = Depends(get_current_user),
):
    ...


@router.delete(
    "/{task_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="Delete a task",
    responses={
        403: {"description": "Only admins can delete tasks"},
        404: {"description": "Task not found"},
    },
)
async def delete_task(
    org_id: UUID,
    project_id: UUID,
    task_id: UUID,
    user: User = Depends(get_current_user),
):
    ...
```

> "Run the app. Open `/docs`. You can see EVERY endpoint, the request/response shapes, the status codes, the error responses. You haven't written a single line of logic, but you can review the entire API. Send this to a frontend developer — they can start building today."

---

## 6.4 From Contract to Implementation Plan

**The contract tells you exactly what to build, in what order:**

```
┌─────────────────────────────────────────────────────────────────┐
│            CONTRACT → IMPLEMENTATION PLAN                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  For EACH router stub, you now need:                            │
│                                                                 │
│  1. SQLAlchemy model (if not yet created)                       │
│     → Translate entity attributes to mapped_column definitions  │
│                                                                 │
│  2. Alembic migration                                           │
│     → alembic revision --autogenerate                           │
│                                                                 │
│  3. Repository methods                                          │
│     → One method per query the service will need                │
│                                                                 │
│  4. Service methods                                             │
│     → One method per router endpoint                            │
│     → Business logic, permission checks, side effects           │
│                                                                 │
│  5. Wire up the router                                          │
│     → Replace ... with service calls                            │
│                                                                 │
│  6. Tests                                                       │
│     → Repository: test against real test DB                     │
│     → Service: test with mocked repository                      │
│     → Router: test with TestClient, full integration            │
│                                                                 │
│                                                                 │
│  Build module by module:                                        │
│  auth → organizations → projects → tasks → notifications →     │
│  audit → attachments                                            │
│                                                                 │
│  Each module: models → migration → repo → service → router →   │
│  tests → next module                                            │
│                                                                 │
│  You now have a ROADMAP. Not just a vague "build the project."  │
│  A concrete, ordered sequence of small, testable steps.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│             ARCHITECTURE QUICK REFERENCE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THE PIPELINE:                                                  │
│  Business brief → Nouns/Verbs → User Stories → Domain Model     │
│  → Module Boundaries → Folder Structure → API Contract → Code   │
│                                                                 │
│  THE LAYERS (outer → inner):                                    │
│  Router → Service → Repository → Model                          │
│                                                                 │
│  THE DEPENDENCY RULE:                                           │
│  Dependencies flow INWARD only. Inner layers never import       │
│  outer layers.                                                  │
│                                                                 │
│  THE ENTITY TEST:                                               │
│  Has identity? Has lifecycle? Referenced elsewhere? → Entity    │
│  Otherwise → Attribute of another entity                        │
│                                                                 │
│  THE MODULE TEST:                                               │
│  "Do these entities change together?"                           │
│  "Can I develop and test this group independently?" → Module    │
│                                                                 │
│  ADR TEMPLATE:                                                  │
│  Status → Context → Decision → Consequences                     │
│  Store in: docs/adr/NNN-title.md                                │
│                                                                 │
│  CONTRACT-FIRST:                                                │
│  Write schemas → Write router stubs → Review in Swagger         │
│  → THEN implement service → repository → tests                  │
│                                                                 │
│  FILE NAMING:                                                   │
│  {module}/router.py    — HTTP endpoints                         │
│  {module}/service.py   — Business logic                         │
│  {module}/repository.py — Database queries                      │
│  {module}/schemas.py   — Pydantic models                        │
│  models/{entity}.py    — SQLAlchemy models (centralized)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ARCHITECTURE = DECISIONS MADE BEFORE CODE                      │
│                                                                 │
│  You spent 12 weeks learning tools:                             │
│  FastAPI, SQLAlchemy, Pydantic, Redis, Celery, WebSockets.      │
│  Those are BRICKS.                                              │
│                                                                 │
│  This lecture is about the BLUEPRINT that tells you              │
│  where each brick goes.                                         │
│                                                                 │
│                                                                 │
│  THE PROCESS:                                                   │
│                                                                 │
│  ┌────────────┐    Read the brief. Extract nouns and verbs.     │
│  │REQUIREMENTS│    Write user stories. Prioritize.              │
│  └─────┬──────┘                                                 │
│        │                                                        │
│        ▼                                                        │
│  ┌────────────┐    Identify entities. Define attributes.        │
│  │  DOMAIN    │    Map relationships. Group into modules.       │
│  │  MODEL     │    This is the THINKING. Don't rush it.         │
│  └─────┬──────┘                                                 │
│        │                                                        │
│        ▼                                                        │
│  ┌────────────┐    Router → Service → Repository → Model.       │
│  │  PROJECT   │    One module per domain. Dependency rule.       │
│  │  STRUCTURE │    Consistency IS the architecture.             │
│  └─────┬──────┘                                                 │
│        │                                                        │
│        ▼                                                        │
│  ┌────────────┐    Document the WHY, not just the WHAT.         │
│  │   ADRs     │    Future you is the audience.                  │
│  └─────┬──────┘                                                 │
│        │                                                        │
│        ▼                                                        │
│  ┌────────────┐    Schemas first. Stubs second. Swagger third.  │
│  │   API      │    Implementation LAST.                         │
│  │ CONTRACT   │                                                 │
│  └────────────┘                                                 │
│                                                                 │
│                                                                 │
│  You don't write a line of logic until all five exist.          │
│  The code writes itself when the design is right.               │
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
│  WEEK 13, LECTURE 2: Multi-Tenancy & Advanced Patterns          │
│  └─ You designed the module boundaries and decided on           │
│     row-level tenancy. Next lecture: IMPLEMENT it.              │
│     Middleware that enforces org_id. Repository filters.        │
│     Tests that prove isolation works.                           │
│                                                                 │
│  WEEK 13, LECTURE 3: Integration Patterns                       │
│  └─ Email, file uploads, search, export.                        │
│     Each integration fits into the SERVICE layer.               │
│     The architecture handles it: add a service method,          │
│     call it from the router. The structure absorbs features.    │
│                                                                 │
│  WEEK 13, LECTURE 4: Code Review & Technical Communication      │
│  └─ ADRs become your PR descriptions.                           │
│     The module structure makes code review tractable —           │
│     reviewers focus on ONE module at a time.                    │
│                                                                 │
│  WEEK 14: Building the Capstone                                 │
│  └─ You won't start from zero. You'll start from:              │
│     domain model + folder structure + router stubs + ADRs.      │
│     Implementation is filling in the blanks.                    │
│                                                                 │
│  WEEK 15: Docker & Deployment                                   │
│  └─ The clean separation means your Dockerfile is simple.       │
│     The workers/ package deploys independently from app/.       │
│     12-factor config lives in core/config.py.                   │
│                                                                 │
│  WEEK 16: System Design                                         │
│  └─ The module boundaries you drew today? Those are             │
│     future service boundaries if you ever go to microservices.  │
│     You're already thinking at the system design level.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```