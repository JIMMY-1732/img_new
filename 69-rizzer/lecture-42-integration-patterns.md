# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NAIVE IMPLEMENTATION FIRST                                     │
│  ────────────────────────                                       │
│  Show the broken version. Let them see WHY every pattern        │
│  exists before learning HOW it works.                           │
│                                                                 │
│  DELEGATION PRINCIPLE                                           │
│  ────────────────────                                           │
│  The recurring theme: your backend is a COORDINATOR,            │
│  not an executor. Outsource to specialists.                     │
│                                                                 │
│  SYNTHESIS OVER NOVELTY                                         │
│  ──────────────────────                                         │
│  Most "new" code here is ASSEMBLING tools students already      │
│  know: httpx, Celery, Redis, SQLAlchemy, Pydantic.             │
│  The lecture teaches how to COMPOSE them into real features.     │
│                                                                 │
│  CONNECT TO CAPSTONE                                            │
│  ────────────────────                                           │
│  Every pattern maps directly to a capstone requirement.         │
│  Students should leave knowing exactly where each piece         │
│  fits in their project.                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                   INTEGRATION PATTERNS                          │
│                     (3-4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE INTEGRATION REALITY (30 min)                       │
│  ├─ 1.1 The Demo That Fell Apart                                │
│  ├─ 1.2 The Core Principle: Delegate, Don't Execute             │
│  ├─ 1.3 The Company Headquarters Analogy                        │
│  └─ 1.4 Where Each Pattern Lives (Project Structure)            │
│                                                                 │
│  PART 2: EMAIL & NOTIFICATION INTEGRATION (45 min)              │
│  ├─ 2.1 The Synchronous Email Trap                              │
│  ├─ 2.2 The Architecture: Endpoint → Task → Provider            │
│  ├─ 2.3 Building the Email Service Abstraction                  │
│  ├─ 2.4 Sending via Background Task                             │
│  ├─ 2.5 Receiving Delivery Status (Webhooks)                    │
│  └─ 2.6 Common Mistakes                                         │
│                                                                 │
│  PART 3: FILE HANDLING WITH OBJECT STORAGE (50 min)             │
│  ├─ 3.1 Where Do Files Live? (The Wrong Answers)                │
│  ├─ 3.2 Object Storage Mental Model                             │
│  ├─ 3.3 MinIO: S3-Compatible Local Development                  │
│  ├─ 3.4 The Presigned URL Pattern                               │
│  ├─ 3.5 File Metadata in Your Database                          │
│  ├─ 3.6 Implementation: Upload & Download                       │
│  └─ 3.7 Common Mistakes                                         │
│                                                                 │
│  PART 4: SEARCH & EXPORT (50 min)                               │
│  ├─ 4.1 The LIKE '%query%' Disaster                             │
│  ├─ 4.2 PostgreSQL Full-Text Search in SQLAlchemy               │
│  ├─ 4.3 The Search Endpoint                                     │
│  ├─ 4.4 When You'd Need a Dedicated Search Engine               │
│  ├─ 4.5 Export: The Request That Never Returns                   │
│  ├─ 4.6 The Async Export Pipeline                                │
│  └─ 4.7 Common Mistakes                                         │
│                                                                 │
│  PART 5: ADMIN DASHBOARD & AGGREGATION (30 min)                 │
│  ├─ 5.1 The Dashboard That Killed the Database                  │
│  ├─ 5.2 Aggregation Queries with CTEs                           │
│  ├─ 5.3 Caching Dashboard Data                                  │
│  └─ 5.4 The Dashboard Endpoint                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE INTEGRATION REALITY

## 1.1 The Demo That Fell Apart

**Set the scene. Every student will recognize this moment.**

> "You've built your capstone backend. CRUD works. Auth works. Real-time works. You demo it to a stakeholder. They say: *'Looks great. Can users get email notifications when tasks are assigned? Can they attach files? Can I search across all projects? Can we export data for compliance? And I need an admin dashboard with stats.'*"

**You say "sure" and start coding naively:**

```python
# ❌ The naive implementation of ALL FIVE features
# (This is what breaks in production)

from fastapi import APIRouter, UploadFile
import smtplib
import csv
import io

router = APIRouter()


# ❌ PROBLEM 1: Email blocks the response for 3-5 seconds
@router.post("/tasks/{task_id}/assign")
async def assign_task(task_id: uuid.UUID, assignee_id: uuid.UUID):
    task = await task_repo.assign(task_id, assignee_id)
    
    # Sending email RIGHT HERE in the handler
    user = await user_repo.get(assignee_id)
    smtp = smtplib.SMTP("smtp.gmail.com", 587)  # ← Blocks the event loop!
    smtp.starttls()
    smtp.login("you@gmail.com", "password123")   # ← Credentials hardcoded!
    smtp.sendmail(
        "you@gmail.com",
        user.email,
        f"Subject: New Task\n\nYou were assigned: {task.title}"
    )
    smtp.quit()
    
    return task  # User waited 4 seconds for this response


# ❌ PROBLEM 2: File stored as BLOB in PostgreSQL
@router.post("/tasks/{task_id}/attachments")
async def upload_attachment(task_id: uuid.UUID, file: UploadFile):
    contents = await file.read()  # ← Entire file in memory
    
    # Storing raw bytes in a PostgreSQL column
    await db.execute(
        "INSERT INTO attachments (task_id, data, filename) VALUES ($1, $2, $3)",
        task_id, contents, file.filename  # ← 50MB PDF in your database!
    )
    return {"status": "uploaded"}


# ❌ PROBLEM 3: LIKE search on 500,000 rows
@router.get("/search")
async def search_tasks(q: str):
    results = await db.fetch_all(
        "SELECT * FROM tasks WHERE title LIKE $1 OR description LIKE $1",
        f"%{q}%"  # ← Full table scan. Every. Single. Time.
    )
    return results


# ❌ PROBLEM 4: Export times out at 30 seconds
@router.get("/export/tasks")
async def export_tasks(org_id: uuid.UUID):
    tasks = await task_repo.get_all_for_org(org_id)  # ← 100,000 rows
    
    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(["id", "title", "status", "assignee", "created_at"])
    for task in tasks:
        writer.writerow([task.id, task.title, task.status, ...])
    
    return Response(
        content=output.getvalue(),  # ← 50MB CSV built synchronously
        media_type="text/csv",      #    Gateway timeout after 30s
    )


# ❌ PROBLEM 5: Dashboard recomputes on every refresh
@router.get("/admin/dashboard")
async def dashboard(org_id: uuid.UUID):
    # 6 expensive queries, every single page load
    total_tasks = await db.fetch_val("SELECT COUNT(*) FROM tasks WHERE org_id=$1", org_id)
    completed = await db.fetch_val("SELECT COUNT(*) FROM tasks WHERE org_id=$1 AND status='done'", org_id)
    active_users = await db.fetch_val("SELECT COUNT(DISTINCT assignee_id) FROM tasks WHERE ...", org_id)
    # ... 3 more queries ...
    
    return {
        "total_tasks": total_tasks,
        "completed": completed,
        "active_users": active_users,
        # Each dashboard refresh = 6 DB round-trips
        # 50 admins refreshing = 300 queries/minute
    }
```

**Now ask the class:**

> "Five features, five disasters waiting to happen. Can you identify what breaks in each one — and *when* it breaks? It might work with 10 users in development. What happens at 1,000?"

**Let them discuss. Then summarize:**

```
┌─────────────────────────────────────────────────────────────────┐
│             WHAT BREAKS AND WHEN                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EMAIL:   Response takes 4+ seconds. SMTP server down?         │
│           Your entire endpoint fails. User sees 500.            │
│           Blocks the async event loop (smtplib is sync).        │
│                                                                 │
│  FILES:   Database balloons from 500MB to 50GB. Backups take   │
│           hours. A single 100MB upload exhausts server RAM.     │
│           PostgreSQL was never designed for binary storage.      │
│                                                                 │
│  SEARCH:  0.01s with 100 rows. 4.7s with 500,000 rows.        │
│           LIKE '%word%' cannot use indexes. Full table scan.    │
│                                                                 │
│  EXPORT:  Works for 100 tasks. Times out at 10,000.            │
│           Gateway returns 504. User retries. Server overloads.  │
│                                                                 │
│  DASHBOARD: Works for 1 admin. 50 admins refreshing every      │
│           10 seconds = 300 queries/minute of pure aggregation.  │
│           Database CPU at 100%.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.2 The Core Principle: Delegate, Don't Execute

**Every pattern in this lecture follows ONE principle:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│       YOUR BACKEND IS A COORDINATOR, NOT AN EXECUTOR.           │
│                                                                 │
│  ❌ Don't send emails          → Delegate to an email service   │
│  ❌ Don't store files          → Delegate to object storage     │
│  ❌ Don't scan every row       → Delegate to a search index     │
│  ❌ Don't build exports inline → Delegate to a background task  │
│  ❌ Don't recompute stats      → Delegate to a cache layer      │
│                                                                 │
│  Your request handler's job:                                    │
│  1. Validate the request                                        │
│  2. Coordinate the right services                               │
│  3. Return a response FAST                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 The Company Headquarters Analogy

**This analogy frames the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│            YOUR BACKEND = COMPANY HEADQUARTERS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌──────────────────┐                          │
│                    │   YOUR BACKEND   │                          │
│                    │  (Headquarters)  │                          │
│                    └────────┬─────────┘                          │
│                             │                                   │
│          ┌─────────┬────────┼────────┬──────────┐               │
│          │         │        │        │          │               │
│          ▼         ▼        ▼        ▼          ▼               │
│     ┌─────────┐┌───────┐┌──────┐┌────────┐┌─────────┐          │
│     │ COURIER ││WARE-  ││FILING││REPORTS ││EXECUTIVE│          │
│     │ SERVICE ││HOUSE  ││INDEX ││DEPT    ││ BOARD   │          │
│     │         ││       ││      ││        ││         │          │
│     │SendGrid ││  S3/  ││Full- ││Celery  ││ Redis   │          │
│     │ Mailgun ││ MinIO ││Text  ││Export  ││ Cached  │          │
│     │         ││       ││Search││Tasks   ││ Stats   │          │
│     └─────────┘└───────┘└──────┘└────────┘└─────────┘          │
│                                                                 │
│  HQ doesn't deliver mail    → Hires FedEx (SendGrid)           │
│  HQ doesn't store boxes     → Rents storage units (S3)         │
│  HQ doesn't search by hand  → Maintains an index (tsvector)    │
│  HQ doesn't print reports   → Sends to print shop (Celery)     │
│  HQ doesn't recount daily   → Reads the dashboard (Redis)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 Where Each Pattern Lives (Project Structure)

> "In Lecture 1 this week, we defined the project structure for larger FastAPI applications. Here's exactly where each integration pattern fits."

```
┌─────────────────────────────────────────────────────────────────┐
│                   PROJECT STRUCTURE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  app/                                                           │
│  ├── integrations/            ← NEW: External service clients   │
│  │   ├── __init__.py                                            │
│  │   ├── email.py             ← Email service abstraction       │
│  │   └── storage.py           ← S3/MinIO service abstraction    │
│  │                                                              │
│  ├── tasks/                   ← Celery tasks (from Week 11)     │
│  │   ├── __init__.py                                            │
│  │   ├── notifications.py     ← Email sending tasks             │
│  │   └── exports.py           ← Data export tasks               │
│  │                                                              │
│  ├── routers/                                                   │
│  │   ├── webhooks.py          ← Webhook receivers               │
│  │   ├── files.py             ← Upload/download endpoints       │
│  │   ├── search.py            ← Search endpoints                │
│  │   ├── exports.py           ← Export request/status           │
│  │   └── admin.py             ← Dashboard endpoints             │
│  │                                                              │
│  ├── repositories/                                              │
│  │   ├── search.py            ← Full-text search queries        │
│  │   └── analytics.py         ← Aggregation queries             │
│  │                                                              │
│  ├── models/                  ← SQLAlchemy (from Week 6)        │
│  │   ├── file_attachment.py   ← File metadata model             │
│  │   ├── notification_log.py  ← Email tracking model            │
│  │   └── export_job.py        ← Export status tracking          │
│  │                                                              │
│  └── schemas/                 ← Pydantic (from Week 3)          │
│      ├── files.py                                               │
│      ├── notifications.py                                       │
│      ├── exports.py                                             │
│      └── analytics.py                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight:**

> "Notice the `integrations/` directory. This is where external service clients live, isolated behind abstractions. Your routes and tasks never call SendGrid or S3 directly — they call your abstraction. If you switch from SendGrid to Mailgun, you change ONE file."

---

# PART 2: EMAIL & NOTIFICATION INTEGRATION

## 2.1 The Synchronous Email Trap

**Recall the broken code from Part 1. Let's trace exactly why it fails:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE SYNCHRONOUS EMAIL DISASTER                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User clicks "Assign Task"                                      │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────────────────────────────────┐           │
│  │  POST /tasks/{id}/assign                         │           │
│  │                                                  │           │
│  │  1. Update DB (50ms)                   ✅ Fast   │           │
│  │  2. Connect to SMTP server (800ms)     😰 Slow   │           │
│  │  3. TLS handshake (200ms)              😰 Slow   │           │
│  │  4. Send email (1500ms)                😰 Slow   │           │
│  │  5. Wait for confirmation (500ms)      😰 Slow   │           │
│  │  6. Return response                    ✅ Done   │           │
│  │                                                  │           │
│  │  Total: ~3,050ms                                 │           │
│  │  Of which ~50ms was YOUR actual work             │           │
│  └──────────────────────────────────────────────────┘           │
│                                                                 │
│  But that's the HAPPY path. What if:                            │
│                                                                 │
│  • SMTP server is down?        → 30s timeout → 504 Gateway     │
│  • Email address is invalid?   → Bounce after sending → ???     │
│  • Rate limit from provider?   → 429 error → Your endpoint 500 │
│  • smtplib is synchronous?     → Blocks your event loop!       │
│    (Remember Week 1, Lecture 3: blocking calls in async code    │
│     freeze ALL concurrent request handling)                     │
│                                                                 │
│  WORST: The task WAS assigned in the DB, but user sees an       │
│  error because the EMAIL failed. Data is inconsistent.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "The task assignment succeeded. The email failed. Should the user see an error? Should we rollback the assignment? Is sending an email part of the core operation, or a side effect?"

Answer: **It's a side effect. The user's action is 'assign the task.' The email is a notification ABOUT that action. They are different concerns with different failure modes.**

---

## 2.2 The Architecture: Endpoint → Task → Provider

**The correct pattern separates the action from its side effects:**

```
┌─────────────────────────────────────────────────────────────────┐
│           EMAIL NOTIFICATION ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User Request                                                   │
│       │                                                         │
│       ▼                                                         │
│  ┌────────────────┐     ┌─────────────┐     ┌───────────────┐   │
│  │   FastAPI       │     │   Celery     │     │   SendGrid    │   │
│  │   Handler       │────▶│   Worker     │────▶│   API         │   │
│  │                 │     │             │     │               │   │
│  │ 1. Assign task  │     │ 1. Build    │     │ 1. Accept     │   │
│  │ 2. Log to DB    │     │    email    │     │    email      │   │
│  │ 3. Queue task   │     │ 2. Call     │     │ 2. Deliver    │   │
│  │ 4. Return 200   │     │    provider │     │ 3. Report     │   │
│  │    (50ms)       │     │ 3. Update   │     │    status     │   │
│  │                 │     │    log      │     │               │   │
│  └────────────────┘     └─────────────┘     └───────┬───────┘   │
│       │                                             │           │
│       │ Fast response                   Webhook     │           │
│       │ to user                         callback    │           │
│       ▼                                             ▼           │
│  ┌────────────────┐                    ┌───────────────────┐    │
│  │ User sees      │                    │ POST /webhooks/   │    │
│  │ "Task assigned"│                    │ sendgrid          │    │
│  │ immediately    │                    │                   │    │
│  │                │                    │ Update delivery   │    │
│  │                │                    │ status in DB      │    │
│  └────────────────┘                    └───────────────────┘    │
│                                                                 │
│  KEY: The user NEVER waits for the email.                       │
│  The email happens AFTER the response is sent.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map it to the analogy:**

> "You're at HQ. A client says 'assign this to John.' You update the ledger (DB), drop a letter in the outbox (Celery queue), and tell the client 'done.' The mail courier (SendGrid) picks up the outbox later and delivers. If the courier has a flat tire, that's their problem — the assignment is already recorded."

---

## 2.3 Building the Email Service Abstraction

**We use a Protocol (Week 1, Lecture 2: Python patterns) to define the contract. This lets us swap providers without touching business logic.**

```python
# app/integrations/email.py

from typing import Protocol
import httpx
from app.core.config import settings


class EmailService(Protocol):
    """Contract for any email provider.
    
    Why Protocol, not ABC?
    Structural subtyping — any class with this method signature works.
    No inheritance required. (Week 1, Lecture 2: Protocols)
    """

    async def send(
        self,
        to: str,
        subject: str,
        html_body: str,
        idempotency_key: str | None = None,
    ) -> str:
        """Send an email. Return the provider's message ID."""
        ...
```

**Now the SendGrid implementation. Notice: we're using httpx (Week 8), not a SendGrid-specific library. Any REST API is just HTTP calls.**

```python
class SendGridEmailService:
    """SendGrid implementation using httpx.
    
    Why httpx and not the sendgrid Python library?
    1. You already know httpx (Week 8).
    2. It keeps dependencies minimal.
    3. It proves the point: any REST API = HTTP calls.
    """
    
    def __init__(self, api_key: str, from_email: str, from_name: str):
        self.api_key = api_key
        self.from_email = from_email
        self.from_name = from_name
        self.base_url = "https://api.sendgrid.com/v3"
    
    async def send(
        self,
        to: str,
        subject: str,
        html_body: str,
        idempotency_key: str | None = None,
    ) -> str:
        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json",
        }
        
        # Idempotency key prevents duplicate sends on retry
        # (Week 11: idempotent tasks — same concept, different layer)
        if idempotency_key:
            headers["X-Idempotency-Key"] = idempotency_key
        
        payload = {
            "personalizations": [{"to": [{"email": to}]}],
            "from": {"email": self.from_email, "name": self.from_name},
            "subject": subject,
            "content": [{"type": "text/html", "value": html_body}],
        }
        
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/mail/send",
                headers=headers,
                json=payload,
                timeout=10.0,
            )
            response.raise_for_status()
        
        # SendGrid returns message ID in header on 202 Accepted
        return response.headers.get("X-Message-Id", "unknown")
```

**Why the abstraction matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                WHY ABSTRACT THE PROVIDER?                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Today:     EmailService ──▶ SendGridEmailService               │
│  Tomorrow:  EmailService ──▶ MailgunEmailService                │
│  Testing:   EmailService ──▶ FakeEmailService (logs to list)    │
│                                                                 │
│  Your Celery tasks, your routes, your business logic —          │
│  NONE of them know which provider is behind EmailService.       │
│  They call send(). That's it.                                   │
│                                                                 │
│  # In tests:                                                    │
│  class FakeEmailService:                                        │
│      def __init__(self):                                        │
│          self.sent: list[dict] = []                              │
│                                                                 │
│      async def send(self, to, subject, html_body, **kw) -> str: │
│          self.sent.append({"to": to, "subject": subject})       │
│          return "fake-message-id"                                │
│                                                                 │
│  # Override in test via dependency_overrides (Week 4)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Sending via Background Task

**The email goes into a Celery task (Week 11), not in the request handler.**

First, the notification log model — we track every email we send:

```python
# app/models/notification_log.py

class NotificationLog(Base):
    __tablename__ = "notification_logs"
    
    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    recipient_email: Mapped[str] = mapped_column(String(255))
    subject: Mapped[str] = mapped_column(String(500))
    
    # Track lifecycle
    status: Mapped[str] = mapped_column(
        String(20), default="queued"
    )  # queued → sent → delivered | bounced | failed
    
    provider_message_id: Mapped[str | None] = mapped_column(String(255))
    error_message: Mapped[str | None] = mapped_column(Text)
    
    # Tenant isolation (Week 13, Lecture 2)
    organization_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organizations.id"))
    
    created_at: Mapped[datetime] = mapped_column(default=func.now())
    updated_at: Mapped[datetime] = mapped_column(default=func.now(), onupdate=func.now())
```

Now the Celery task:

```python
# app/tasks/notifications.py

from app.core.celery_app import celery_app
from app.integrations.email import SendGridEmailService
from app.core.config import settings
import httpx


@celery_app.task(
    bind=True,
    autoretry_for=(httpx.HTTPStatusError, httpx.ConnectError),
    retry_backoff=True,       # Exponential backoff (Week 11)
    retry_backoff_max=600,    # Cap at 10 minutes
    max_retries=5,
    acks_late=True,           # Only acknowledge AFTER success
)
def send_email_task(
    self,
    notification_log_id: str,
    to: str,
    subject: str,
    html_body: str,
) -> None:
    """Send email via provider. Runs in Celery worker (sync context).
    
    IMPORTANT: Celery tasks run synchronously.
    We use httpx SYNC client here, not async.
    (Week 11: Celery workers have their own process — no event loop)
    """
    import asyncio
    
    email_service = SendGridEmailService(
        api_key=settings.SENDGRID_API_KEY,
        from_email=settings.FROM_EMAIL,
        from_name=settings.FROM_NAME,
    )
    
    # Run the async send in a one-shot event loop
    # (This is one of the few places asyncio.run() inside non-async is correct)
    try:
        message_id = asyncio.run(
            email_service.send(
                to=to,
                subject=subject,
                html_body=html_body,
                idempotency_key=notification_log_id,  # Prevents duplicates on retry
            )
        )
        
        # Update log: sent
        update_notification_status(
            notification_log_id, "sent", provider_message_id=message_id
        )
        
    except httpx.HTTPStatusError as exc:
        # Update log: failed (will be retried by Celery)
        update_notification_status(
            notification_log_id, "failed", error=str(exc)
        )
        raise  # Re-raise so Celery retries
```

**And the endpoint that triggers it all:**

```python
# app/routers/tasks.py

@router.post("/{task_id}/assign", response_model=TaskResponse)
async def assign_task(
    task_id: uuid.UUID,
    body: AssignTaskRequest,
    current_user: User = Depends(get_current_user),
    session: AsyncSession = Depends(get_session),
):
    # 1. Core operation: assign the task
    task = await task_repo.assign(session, task_id, body.assignee_id)
    assignee = await user_repo.get(session, body.assignee_id)
    
    # 2. Log the notification intent
    log = NotificationLog(
        recipient_email=assignee.email,
        subject=f"You've been assigned: {task.title}",
        status="queued",
        organization_id=current_user.organization_id,
    )
    session.add(log)
    await session.commit()
    
    # 3. Queue the email (fire-and-forget from this handler's perspective)
    send_email_task.delay(
        notification_log_id=str(log.id),
        to=assignee.email,
        subject=log.subject,
        html_body=render_assignment_email(task, assignee, current_user),
    )
    
    # 4. Return immediately — user waited ~60ms, not 3 seconds
    return task
```

**The critical timeline comparison:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 BEFORE vs AFTER                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE (synchronous email):                                    │
│                                                                 │
│  Request ──[DB 50ms]──[SMTP 3000ms]──▶ Response                 │
│                                        Total: ~3050ms           │
│                                        If SMTP fails: 500 error │
│                                                                 │
│  AFTER (background task):                                       │
│                                                                 │
│  Request ──[DB 50ms]──[Queue 5ms]──▶ Response                   │
│                            │          Total: ~55ms              │
│                            │          Email failure: invisible  │
│                            ▼            to user                 │
│                     [Celery Worker]                              │
│                     [SMTP 3000ms ]                              │
│                     (async, with retries)                       │
│                                                                 │
│  Response time: 55x faster                                      │
│  Reliability: retries handle transient failures                 │
│  Decoupling: email provider outage ≠ API outage                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.5 Receiving Delivery Status (Webhooks)

> "You sent the email. But did it arrive? SendGrid tells you via webhooks — HTTP callbacks to YOUR server. You learned the concept in Week 8. Now we implement the receiving side."

```python
# app/schemas/notifications.py

class SendGridWebhookEvent(BaseModel):
    """One event from SendGrid's Event Webhook.
    
    SendGrid POSTs a JSON array of these to your endpoint.
    Always validate with Pydantic — never trust external data.
    (Week 8, Lecture 3: external vs internal models)
    """
    email: str
    event: str  # "delivered", "bounce", "dropped", "open", "click"
    sg_message_id: str | None = None
    reason: str | None = None   # Bounce reason
    timestamp: int              # Unix timestamp
    
    model_config = ConfigDict(extra="ignore")  # Ignore fields we don't care about
```

```python
# app/routers/webhooks.py

from fastapi import APIRouter, Request, HTTPException, status
import hashlib
import hmac

router = APIRouter(prefix="/webhooks", tags=["webhooks"])


def verify_sendgrid_signature(request: Request, body: bytes) -> bool:
    """Verify the webhook actually came from SendGrid, not an attacker.
    
    Same principle as JWT signature verification (Week 9):
    The sender signs the payload with a shared secret.
    We verify the signature before trusting the data.
    """
    signature = request.headers.get("X-Twilio-Email-Event-Webhook-Signature", "")
    timestamp = request.headers.get("X-Twilio-Email-Event-Webhook-Timestamp", "")
    
    verification_key = settings.SENDGRID_WEBHOOK_VERIFICATION_KEY
    
    # Construct the signed payload
    payload = timestamp.encode() + body
    expected = hmac.new(
        verification_key.encode(), payload, hashlib.sha256
    ).hexdigest()
    
    return hmac.compare_digest(signature, expected)


@router.post("/sendgrid", status_code=status.HTTP_200_OK)
async def handle_sendgrid_webhook(
    request: Request,
    session: AsyncSession = Depends(get_session),
):
    body = await request.body()
    
    if not verify_sendgrid_signature(request, body):
        raise HTTPException(status_code=401, detail="Invalid signature")
    
    import json
    events = [SendGridWebhookEvent(**e) for e in json.loads(body)]
    
    for event in events:
        if event.event == "delivered":
            await notification_repo.update_status(
                session, event.sg_message_id, "delivered"
            )
        elif event.event in ("bounce", "dropped"):
            await notification_repo.update_status(
                session, event.sg_message_id, "bounced",
                error=event.reason,
            )
    
    await session.commit()
    
    # ALWAYS return 200 quickly.
    # If you return an error, SendGrid will retry — you'll get duplicates.
    return {"status": "processed", "count": len(events)}
```

**The full webhook lifecycle:**

```
┌─────────────────────────────────────────────────────────────────┐
│                WEBHOOK LIFECYCLE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOUR APP                    SENDGRID                           │
│  ────────                    ────────                           │
│      │                                                          │
│      │── POST /v3/mail/send ────────▶│                          │
│      │                               │                          │
│      │◀─────── 202 Accepted ─────────│                          │
│      │   (message_id: "abc123")      │                          │
│      │                               │                          │
│      │                               │── Delivers email ──▶ 📧  │
│      │                               │                          │
│      │                               │                          │
│      │◀── POST /webhooks/sendgrid ───│                          │
│      │    [{                         │                          │
│      │      "event": "delivered",    │                          │
│      │      "sg_message_id": "abc",  │                          │
│      │      "timestamp": 170000000   │                          │
│      │    }]                         │                          │
│      │                               │                          │
│      │─── 200 OK ──────────────────▶│                          │
│      │                               │                          │
│      │   (Update notification_logs:                             │
│      │    status = "delivered")                                  │
│      │                                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.6 Common Mistakes

```
┌─────────────────────────────────────────────────────────────────┐
│              EMAIL INTEGRATION MISTAKES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ MISTAKE 1: Sending email in the request handler             │
│     Fix: Always use a background task (Celery)                  │
│                                                                 │
│  ❌ MISTAKE 2: No idempotency key on retries                    │
│     Result: User gets 5 identical emails when Celery retries    │
│     Fix: Pass a unique key (notification_log_id) to provider    │
│                                                                 │
│  ❌ MISTAKE 3: Returning error from webhook endpoint            │
│     Result: Provider retries → you process same event 10 times  │
│     Fix: Always return 200, handle errors internally            │
│                                                                 │
│  ❌ MISTAKE 4: Not verifying webhook signatures                 │
│     Result: Attacker can forge "delivered" events               │
│     Fix: HMAC verification on every webhook request             │
│                                                                 │
│  ❌ MISTAKE 5: Hardcoding provider in business logic            │
│     Result: Switching providers = rewriting half the codebase   │
│     Fix: Protocol abstraction, swap via dependency injection    │
│                                                                 │
│  ❌ MISTAKE 6: Not logging email attempts                       │
│     Result: "Did the email send?" → Shrug.                      │
│     Fix: notification_logs table tracks every attempt + status  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: FILE HANDLING WITH OBJECT STORAGE

## 3.1 Where Do Files Live? (The Wrong Answers)

**Ask the class:**

> "A user wants to attach a PDF to their task. Where do you store the file? You have 3 seconds."

**The three wrong answers students always give:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THREE WRONG ANSWERS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ ANSWER 1: "In a BYTEA column in PostgreSQL"                 │
│                                                                 │
│     Problems:                                                   │
│     • 10,000 tasks × 5MB avg attachment = 50GB in your DB      │
│     • pg_dump backup takes hours instead of minutes             │
│     • Every query on the tasks table gets slower                │
│     • Database connection held open during entire download      │
│     • PostgreSQL max row size complications                     │
│                                                                 │
│  ❌ ANSWER 2: "On the server filesystem (/uploads/...)"         │
│                                                                 │
│     Problems:                                                   │
│     • Server gets replaced during deployment → files gone       │
│     • Multiple servers? Files only on one of them               │
│     • Disk fills up → entire server crashes                     │
│     • No CDN, no geographic distribution                        │
│     • Backup = copy entire filesystem (slow, fragile)           │
│                                                                 │
│  ❌ ANSWER 3: "Base64 encode and put in a JSON field"           │
│                                                                 │
│     Problems:                                                   │
│     • 33% size overhead (5MB file → 6.7MB stored)              │
│     • Every API response with the file = massive JSON          │
│     • Parsing/serializing megabytes of Base64 = CPU burn       │
│     • All the PostgreSQL problems from Answer 1, but worse     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Object Storage Mental Model

**The right answer: Object Storage (S3, MinIO, GCS, Azure Blob).**

```
┌─────────────────────────────────────────────────────────────────┐
│                  OBJECT STORAGE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Think of it as an infinite, organized filing cabinet           │
│  that lives OUTSIDE your application:                           │
│                                                                 │
│  CONCEPTS:                                                      │
│  ─────────                                                      │
│  Bucket   = A named container (like a filing cabinet)           │
│             e.g., "my-saas-attachments"                         │
│                                                                 │
│  Object   = A file stored in a bucket                           │
│             Has a KEY (path) and the DATA (bytes)               │
│             e.g., key = "orgs/abc/tasks/123/report.pdf"         │
│                                                                 │
│  Key      = The "path" to an object (like a file path)          │
│             NOT a real filesystem — just a string               │
│             Slashes are convention, not directories              │
│                                                                 │
│  Metadata = Info about the object (content-type, size, etc.)    │
│                                                                 │
│                                                                 │
│  WHY IT'S BETTER:                                               │
│  ────────────────                                               │
│  • Designed for binary data (TB-scale, not MB-scale)            │
│  • Independent scaling (storage grows without affecting DB)     │
│  • Built-in redundancy (99.999999999% durability for S3)        │
│  • CDN integration (serve files from edge locations)            │
│  • Access control via signed URLs (no auth on your server)      │
│  • Pay per GB stored + GB transferred (cents, not dollars)      │
│                                                                 │
│                                                                 │
│  YOUR DATABASE STORES:     OBJECT STORAGE STORES:               │
│  ─────────────────────     ──────────────────────               │
│  File metadata             File bytes                           │
│  • id (UUID)               • The actual PDF, image, etc.        │
│  • filename                                                     │
│  • content_type                                                 │
│  • size_bytes                                                   │
│  • storage_key ────────────────▶ Used to locate the object      │
│  • uploaded_by                                                  │
│  • created_at                                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 MinIO: S3-Compatible Local Development

> "You know Docker from Week 5 (running PostgreSQL) and Week 10 (running Redis). MinIO is another container — an S3-compatible object store you run locally."

**Add to your Docker Compose:**

```yaml
# docker-compose.yml (extend your existing file)
services:
  # ... postgres, redis, celery worker from previous weeks ...
  
  minio:
    image: minio/minio
    ports:
      - "9000:9000"   # S3 API
      - "9001:9001"   # Web console (like pgAdmin but for files)
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data

volumes:
  minio_data:
```

**The beauty: boto3 works identically with MinIO and real AWS S3.**

```python
# app/integrations/storage.py

import boto3
from botocore.config import Config
from app.core.config import settings


def get_s3_client():
    """Create S3 client. Points to MinIO locally, AWS S3 in production.
    
    The ONLY difference between local and production:
    - endpoint_url: http://localhost:9000 vs None (uses AWS default)
    - credentials: minioadmin vs real AWS IAM credentials
    
    Your application code doesn't change AT ALL.
    """
    return boto3.client(
        "s3",
        endpoint_url=settings.S3_ENDPOINT_URL,       # MinIO locally, None for AWS
        aws_access_key_id=settings.S3_ACCESS_KEY,
        aws_secret_access_key=settings.S3_SECRET_KEY,
        region_name=settings.S3_REGION,
        config=Config(signature_version="s3v4"),
    )
```

```
┌─────────────────────────────────────────────────────────────────┐
│          LOCAL vs PRODUCTION — SAME CODE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LOCAL (.env):                                                  │
│  S3_ENDPOINT_URL=http://localhost:9000                           │
│  S3_ACCESS_KEY=minioadmin                                       │
│  S3_SECRET_KEY=minioadmin                                       │
│  S3_BUCKET=my-saas-dev                                          │
│                                                                 │
│  PRODUCTION (.env):                                             │
│  S3_ENDPOINT_URL=              ← Empty: uses AWS default        │
│  S3_ACCESS_KEY=AKIA...         ← Real IAM credentials           │
│  S3_SECRET_KEY=wJal...                                          │
│  S3_BUCKET=my-saas-prod                                         │
│                                                                 │
│  Your Python code: IDENTICAL in both environments.              │
│  Config via pydantic-settings (Week 15, Lecture 2).             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 The Presigned URL Pattern

**This is the most important concept in this section. Read carefully.**

> "What if I told you the file bytes should NEVER pass through your FastAPI server? The client uploads DIRECTLY to S3. Your server just issues a permission slip."

```
┌─────────────────────────────────────────────────────────────────┐
│              TWO UPLOAD APPROACHES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ THROUGH YOUR SERVER (simple, but doesn't scale):            │
│                                                                 │
│  Client ──[50MB file]──▶ FastAPI ──[50MB file]──▶ S3            │
│                           │                                     │
│                           ├─ 50MB in server RAM                 │
│                           ├─ Connection held for upload duration │
│                           ├─ Server bandwidth = bottleneck      │
│                           └─ 10 simultaneous uploads = crash    │
│                                                                 │
│                                                                 │
│  ✅ PRESIGNED URL (production pattern):                         │
│                                                                 │
│  Client ──[request]──▶ FastAPI ──[generate URL]──▶ Client       │
│                         (5ms)                        │          │
│                                                      │          │
│            Client ──[50MB file]──────────────────▶ S3 directly  │
│                     (your server never sees the bytes)          │
│                                                                 │
│  Server work: Generate a signed URL. That's it.                 │
│  No RAM pressure. No bandwidth bottleneck. Scales infinitely.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What IS a presigned URL?**

```
┌─────────────────────────────────────────────────────────────────┐
│                  PRESIGNED URL ANATOMY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A presigned URL is a normal S3 URL with a cryptographic        │
│  signature appended. It says:                                   │
│                                                                 │
│  "The holder of this URL is allowed to PUT a file               │
│   to this exact key, with this exact content type,              │
│   for the next 15 minutes."                                     │
│                                                                 │
│  https://my-bucket.s3.amazonaws.com/orgs/abc/file.pdf           │
│    ?X-Amz-Algorithm=AWS4-HMAC-SHA256                            │
│    &X-Amz-Credential=AKIA.../20260214/us-east-1/s3/...         │
│    &X-Amz-Date=20260214T100000Z                                 │
│    &X-Amz-Expires=900             ← 15 minutes                  │
│    &X-Amz-Signature=a8b3c...     ← Cryptographic signature     │
│                                                                 │
│  PROPERTIES:                                                    │
│  • Time-limited (expires after N seconds)                       │
│  • Operation-specific (PUT only, or GET only)                   │
│  • Key-specific (can only write to the specified path)          │
│  • Signed with YOUR credentials (but doesn't expose them)       │
│  • Generated LOCALLY — no network call to AWS!                  │
│                                                                 │
│  ANALOGY: It's like a JWT (Week 9) for file operations.         │
│  Signed, time-limited, scoped to a specific action.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 File Metadata in Your Database

**Your database tracks files. S3 stores files. They're linked by the storage key.**

```python
# app/models/file_attachment.py

class FileAttachment(Base):
    __tablename__ = "file_attachments"
    
    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    
    # What the user sees
    original_filename: Mapped[str] = mapped_column(String(255))
    content_type: Mapped[str] = mapped_column(String(100))
    size_bytes: Mapped[int] = mapped_column(BigInteger)
    
    # Where it lives in S3 — THE CRITICAL LINK
    storage_key: Mapped[str] = mapped_column(String(500), unique=True)
    # e.g., "orgs/abc123/tasks/def456/2026-02-14_report.pdf"
    
    # Upload lifecycle
    upload_status: Mapped[str] = mapped_column(
        String(20), default="pending"
    )  # pending → confirmed → deleted
    
    # Ownership and tenant isolation (Week 13, Lecture 2)
    uploaded_by: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"))
    organization_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organizations.id"))
    
    # Polymorphic attachment: which entity is this attached to?
    attached_to_type: Mapped[str] = mapped_column(String(50))   # "task", "comment"
    attached_to_id: Mapped[uuid.UUID] = mapped_column()
    
    created_at: Mapped[datetime] = mapped_column(default=func.now())
```

```
┌─────────────────────────────────────────────────────────────────┐
│               DB vs S3 SEPARATION                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   PostgreSQL (file_attachments table)                           │
│   ┌────────────┬───────────┬──────────────────────────────┐     │
│   │ id         │ filename  │ storage_key                  │     │
│   ├────────────┼───────────┼──────────────────────────────┤     │
│   │ aaa-111    │ spec.pdf  │ orgs/xyz/tasks/1/spec.pdf   │──┐  │
│   │ bbb-222    │ logo.png  │ orgs/xyz/tasks/2/logo.png   │──┼┐ │
│   └────────────┴───────────┴──────────────────────────────┘  ││ │
│                                                               ││ │
│   S3 Bucket (my-saas-attachments)                            ││ │
│   ┌──────────────────────────────────────┐                   ││ │
│   │ orgs/xyz/tasks/1/spec.pdf  [4.2MB] ◀┘│                   │ │
│   │ orgs/xyz/tasks/2/logo.png  [150KB] ◀──┘                  │ │
│   └──────────────────────────────────────┘                    │ │
│                                                               │ │
│   storage_key is the bridge between your DB and S3.           │ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why `upload_status`?**

> "There's a gap between 'we generated the presigned URL' and 'the client actually uploaded the file.' The `pending` status means we issued a URL but haven't confirmed the upload arrived. This prevents phantom records in your DB for uploads that never completed."

---

## 3.6 Implementation: Upload & Download

**The storage service abstraction:**

```python
# app/integrations/storage.py

class StorageService:
    """Wraps S3/MinIO operations behind a clean interface."""
    
    def __init__(self, s3_client, bucket_name: str):
        self.s3 = s3_client
        self.bucket = bucket_name
    
    def generate_upload_url(
        self,
        key: str,
        content_type: str,
        expires_in: int = 900,  # 15 minutes
    ) -> str:
        """Generate a presigned URL for uploading.
        
        This is a LOCAL computation — no network call.
        It signs the request parameters with your credentials.
        Safe to call from an async handler without blocking.
        """
        return self.s3.generate_presigned_url(
            ClientMethod="put_object",
            Params={
                "Bucket": self.bucket,
                "Key": key,
                "ContentType": content_type,
            },
            ExpiresIn=expires_in,
        )
    
    def generate_download_url(
        self,
        key: str,
        expires_in: int = 3600,  # 1 hour
        filename: str | None = None,
    ) -> str:
        """Generate a presigned URL for downloading."""
        params = {"Bucket": self.bucket, "Key": key}
        
        if filename:
            # Forces browser to download with original filename
            params["ResponseContentDisposition"] = f'attachment; filename="{filename}"'
        
        return self.s3.generate_presigned_url(
            ClientMethod="get_object",
            Params=params,
            ExpiresIn=expires_in,
        )
    
    def check_object_exists(self, key: str) -> bool:
        """Verify an object was actually uploaded."""
        try:
            self.s3.head_object(Bucket=self.bucket, Key=key)
            return True
        except self.s3.exceptions.ClientError:
            return False
```

**The upload endpoint:**

```python
# app/routers/files.py

router = APIRouter(prefix="/files", tags=["files"])

# Allowed types and size limits — your security boundary
ALLOWED_CONTENT_TYPES = {
    "application/pdf",
    "image/png",
    "image/jpeg",
    "text/csv",
}
MAX_FILE_SIZE = 50 * 1024 * 1024  # 50MB


class UploadRequest(BaseModel):
    filename: str = Field(min_length=1, max_length=255)
    content_type: str
    size_bytes: int = Field(gt=0)
    attached_to_type: str = Field(pattern=r"^(task|comment|project)$")
    attached_to_id: uuid.UUID


class UploadResponse(BaseModel):
    file_id: uuid.UUID
    upload_url: str
    expires_in: int


@router.post("/upload-url", response_model=UploadResponse, status_code=201)
async def request_upload_url(
    body: UploadRequest,
    current_user: User = Depends(get_current_user),
    session: AsyncSession = Depends(get_session),
    storage: StorageService = Depends(get_storage_service),
):
    """Step 1: Client requests a presigned upload URL.
    
    This does NOT upload the file. It gives the client
    a time-limited, signed URL to upload DIRECTLY to S3.
    """
    # Validate content type
    if body.content_type not in ALLOWED_CONTENT_TYPES:
        raise HTTPException(
            status_code=422,
            detail=f"Content type '{body.content_type}' not allowed. "
                   f"Allowed: {ALLOWED_CONTENT_TYPES}"
        )
    
    # Validate size
    if body.size_bytes > MAX_FILE_SIZE:
        raise HTTPException(
            status_code=422,
            detail=f"File too large. Maximum: {MAX_FILE_SIZE // 1024 // 1024}MB"
        )
    
    # Build the storage key — tenant-isolated path
    storage_key = (
        f"orgs/{current_user.organization_id}/"
        f"{body.attached_to_type}s/{body.attached_to_id}/"
        f"{uuid.uuid4().hex[:8]}_{body.filename}"
    )
    
    # Create DB record (status: pending)
    file_record = FileAttachment(
        original_filename=body.filename,
        content_type=body.content_type,
        size_bytes=body.size_bytes,
        storage_key=storage_key,
        upload_status="pending",
        uploaded_by=current_user.id,
        organization_id=current_user.organization_id,
        attached_to_type=body.attached_to_type,
        attached_to_id=body.attached_to_id,
    )
    session.add(file_record)
    await session.commit()
    
    # Generate presigned URL (local computation, fast)
    expires_in = 900
    upload_url = storage.generate_upload_url(
        key=storage_key,
        content_type=body.content_type,
        expires_in=expires_in,
    )
    
    return UploadResponse(
        file_id=file_record.id,
        upload_url=upload_url,
        expires_in=expires_in,
    )
```

**The confirmation endpoint:**

```python
@router.post("/{file_id}/confirm", response_model=FileResponse)
async def confirm_upload(
    file_id: uuid.UUID,
    current_user: User = Depends(get_current_user),
    session: AsyncSession = Depends(get_session),
    storage: StorageService = Depends(get_storage_service),
):
    """Step 3: Client confirms the upload completed.
    
    We verify the object actually exists in S3 before
    marking the record as confirmed. Trust but verify.
    """
    file_record = await file_repo.get_by_id_and_org(
        session, file_id, current_user.organization_id
    )
    if not file_record:
        raise HTTPException(status_code=404, detail="File not found")
    
    if file_record.upload_status != "pending":
        raise HTTPException(status_code=409, detail="File already confirmed")
    
    # Verify the file actually made it to S3
    if not storage.check_object_exists(file_record.storage_key):
        raise HTTPException(
            status_code=400,
            detail="File not found in storage. Upload may have failed."
        )
    
    file_record.upload_status = "confirmed"
    await session.commit()
    
    return file_record
```

**The download endpoint:**

```python
@router.get("/{file_id}/download")
async def get_download_url(
    file_id: uuid.UUID,
    current_user: User = Depends(get_current_user),
    session: AsyncSession = Depends(get_session),
    storage: StorageService = Depends(get_storage_service),
):
    """Generate a time-limited download URL.
    
    Client receives the URL, then fetches the file DIRECTLY
    from S3. Our server never touches the file bytes.
    """
    file_record = await file_repo.get_by_id_and_org(
        session, file_id, current_user.organization_id
    )
    if not file_record or file_record.upload_status != "confirmed":
        raise HTTPException(status_code=404, detail="File not found")
    
    download_url = storage.generate_download_url(
        key=file_record.storage_key,
        filename=file_record.original_filename,
    )
    
    return {"download_url": download_url, "expires_in": 3600}
```

**The complete upload-download flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  FULL FILE LIFECYCLE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   CLIENT                 YOUR API              S3 / MINIO       │
│   ──────                 ────────              ─────────        │
│      │                      │                      │            │
│  UPLOAD:                    │                      │            │
│  ────────                   │                      │            │
│      │                      │                      │            │
│   1. │── POST /files/       │                      │            │
│      │   upload-url ───────▶│                      │            │
│      │   {filename,         │ Validate             │            │
│      │    content_type,     │ Create DB record     │            │
│      │    size_bytes}       │ Generate signed URL   │            │
│      │                      │                      │            │
│      │◀── 201 ──────────────│                      │            │
│      │   {file_id,          │                      │            │
│      │    upload_url}       │                      │            │
│      │                      │                      │            │
│   2. │── PUT upload_url ────────────────────────▶│            │
│      │   [file bytes]       │                    │ Store file  │
│      │                      │                    │             │
│      │◀── 200 ──────────────────────────────────│             │
│      │                      │                      │            │
│   3. │── POST /files/       │                      │            │
│      │   {file_id}/confirm ▶│                      │            │
│      │                      │── HEAD object ──────▶│            │
│      │                      │◀── 200 (exists) ────│            │
│      │                      │ Mark "confirmed"     │            │
│      │◀── 200 ──────────────│                      │            │
│      │                      │                      │            │
│  DOWNLOAD:                  │                      │            │
│  ──────────                 │                      │            │
│      │                      │                      │            │
│   4. │── GET /files/        │                      │            │
│      │   {file_id}/download▶│                      │            │
│      │                      │ Check permissions    │            │
│      │                      │ Generate signed URL   │            │
│      │◀── 200 ──────────────│                      │            │
│      │   {download_url}     │                      │            │
│      │                      │                      │            │
│   5. │── GET download_url ─────────────────────▶│            │
│      │                      │                    │ Serve file  │
│      │◀── 200 [file bytes] ────────────────────│             │
│      │                      │                      │            │
│                                                                 │
│  Steps 2 and 5: Your server is NOT involved.                    │
│  The heavy lifting (file transfer) goes directly to/from S3.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.7 Common Mistakes

```
┌─────────────────────────────────────────────────────────────────┐
│               FILE HANDLING MISTAKES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ MISTAKE 1: Storing file bytes in PostgreSQL                 │
│     Fix: Use object storage. DB stores metadata only.           │
│                                                                 │
│  ❌ MISTAKE 2: Not validating content type                      │
│     Result: Users upload .exe files disguised as .pdf           │
│     Fix: Allowlist of content types, validate BEFORE giving URL │
│                                                                 │
│  ❌ MISTAKE 3: Predictable storage keys                         │
│     "attachments/1.pdf", "attachments/2.pdf"                    │
│     Result: Anyone can guess and enumerate files                │
│     Fix: Include random prefix (uuid hex) in key               │
│                                                                 │
│  ❌ MISTAKE 4: No confirmation step                             │
│     Result: DB has records for uploads that never completed     │
│     Fix: pending → confirmed lifecycle with HEAD check          │
│     ALSO: Periodic cleanup job (Celery Beat, Week 11)           │
│           to delete stale "pending" records after 24h           │
│                                                                 │
│  ❌ MISTAKE 5: Presigned URLs with no expiration limit          │
│     Fix: Upload URLs: 15 min max. Download: 1 hour max.        │
│     A leaked URL becomes useless once expired.                  │
│                                                                 │
│  ❌ MISTAKE 6: Not isolating storage keys by tenant             │
│     Result: Org A could potentially access Org B's files        │
│     Fix: Key structure includes org_id:                         │
│          "orgs/{org_id}/tasks/{task_id}/{filename}"             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: SEARCH & EXPORT

## 4.1 The LIKE '%query%' Disaster

**Start with a demonstration of the scale problem:**

```sql
-- Your table has 500,000 tasks across all tenants.
-- User searches for "quarterly report"

-- ❌ The naive query:
SELECT * FROM tasks
WHERE organization_id = 'abc-123'
  AND (title LIKE '%quarterly report%' 
       OR description LIKE '%quarterly report%');
```

**Run EXPLAIN ANALYZE (Week 7):**

```
┌─────────────────────────────────────────────────────────────────┐
│              LIKE '%...%' EXPLAIN RESULTS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Seq Scan on tasks  (cost=0.00..28541.00 rows=250 width=384)   │
│    Filter: ((title ~~ '%quarterly report%') OR                  │
│             (description ~~ '%quarterly report%'))              │
│    Rows Removed by Filter: 499750                               │
│  Planning Time: 0.15 ms                                         │
│  Execution Time: 4,723.41 ms    ← Almost 5 seconds!            │
│                                                                 │
│  WHY: Leading wildcard (%) prevents index use.                  │
│  PostgreSQL must read EVERY ROW and check the text.             │
│  This is O(n) — gets worse linearly with table size.            │
│                                                                 │
│  At 100 rows:     0.01s  ← "Works fine in development"         │
│  At 10,000 rows:  0.09s  ← "Still fast enough"                 │
│  At 100,000 rows: 0.94s  ← "Getting slow..."                   │
│  At 500,000 rows: 4.72s  ← "Users are leaving"                 │
│                                                                 │
│  Remember Week 7: "EXPLAIN before you ship."                    │
│  LIKE '%...%' is the most common query performance trap.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 PostgreSQL Full-Text Search in SQLAlchemy

> "In Week 5, Lecture 3, you learned about tsvector and tsquery in raw SQL. In Week 7, you learned about GIN indexes. Now we bring them together in SQLAlchemy for production use."

**The mental model: full-text search is an inverted index for text.**

```
┌─────────────────────────────────────────────────────────────────┐
│          LIKE vs FULL-TEXT SEARCH                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LIKE '%quarterly report%':                                     │
│  ──────────────────────────                                     │
│  Read every row → check if text contains substring              │
│  Like reading every page of every book to find a phrase.        │
│                                                                 │
│  FULL-TEXT SEARCH:                                              │
│  ─────────────────                                              │
│  Pre-built index maps words → rows that contain them            │
│  Like using the index at the back of a textbook.                │
│                                                                 │
│  tsvector:  Preprocessed, indexed representation of text        │
│             "Quarterly Budget Report" →                         │
│             'budget':2 'quarter':1 'report':3                   │
│             (stemmed: "quarterly" → "quarter")                  │
│                                                                 │
│  tsquery:   Your search query in the same format                │
│             "quarterly report" → 'quarter' & 'report'           │
│                                                                 │
│  GIN Index: Pre-computed lookup table                           │
│             'budget'  → [row 45, row 1203, row 8891]           │
│             'quarter' → [row 45, row 302, row 7721]            │
│             'report'  → [row 45, row 1203, row 5502]           │
│                                                                 │
│  Search: 'quarter' & 'report' → intersect sets → row 45        │
│  Time: O(log n) — barely changes with table size                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Adding full-text search to your SQLAlchemy model:**

```python
# app/models/task.py — Add search_vector to existing model

from sqlalchemy import Index, String, Text, func
from sqlalchemy.dialects.postgresql import TSVECTOR
from sqlalchemy.schema import Computed


class Task(Base):
    __tablename__ = "tasks"
    
    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    title: Mapped[str] = mapped_column(String(255))
    description: Mapped[str | None] = mapped_column(Text)
    organization_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organizations.id"))
    # ... other columns from your existing model ...
    
    # ── NEW: Generated tsvector column ──
    # PostgreSQL computes this automatically when title/description change.
    # You never set it manually. It's DERIVED data.
    search_vector: Mapped[str | None] = mapped_column(
        TSVECTOR,
        Computed(
            "to_tsvector('english', coalesce(title, '') || ' ' || coalesce(description, ''))",
            persisted=True,  # Stored on disk, not recomputed per query
        ),
    )
    
    __table_args__ = (
        # GIN index on the tsvector column (Week 7: index types)
        # Without this index, full-text search still does a sequential scan!
        Index(
            "ix_tasks_search_vector",
            "search_vector",
            postgresql_using="gin",
        ),
        # Keep your existing indexes...
    )
```

**The Alembic migration for this (Week 6):**

```python
# After running: alembic revision --autogenerate -m "add full-text search to tasks"
# Review the generated migration carefully:

def upgrade():
    op.add_column(
        "tasks",
        sa.Column(
            "search_vector",
            TSVECTOR,
            sa.Computed(
                "to_tsvector('english', coalesce(title, '') || ' ' || coalesce(description, ''))",
                persisted=True,
            ),
        ),
    )
    op.create_index(
        "ix_tasks_search_vector",
        "tasks",
        ["search_vector"],
        postgresql_using="gin",
    )

def downgrade():
    op.drop_index("ix_tasks_search_vector")
    op.drop_column("tasks", "search_vector")
```

**The search repository:**

```python
# app/repositories/search.py

from sqlalchemy import select, func, desc
from sqlalchemy.ext.asyncio import AsyncSession


async def search_tasks(
    session: AsyncSession,
    query: str,
    organization_id: uuid.UUID,
    limit: int = 20,
    offset: int = 0,
) -> tuple[list[Task], int]:
    """Full-text search across task titles and descriptions.
    
    plainto_tsquery: Converts plain text to tsquery.
      "quarterly report" → 'quarter' & 'report'
      Handles stemming, stop words, and spaces automatically.
      Much safer than raw tsquery (no syntax errors from user input).
    
    ts_rank: Scores how well each row matches the query.
      Higher rank = more relevant. Used for ORDER BY.
    """
    search_query = func.plainto_tsquery("english", query)
    
    # Count total matches (for pagination metadata)
    count_stmt = (
        select(func.count())
        .select_from(Task)
        .where(Task.organization_id == organization_id)
        .where(Task.search_vector.match(query))
    )
    total = (await session.execute(count_stmt)).scalar() or 0
    
    # Fetch ranked results
    results_stmt = (
        select(Task)
        .where(Task.organization_id == organization_id)
        .where(Task.search_vector.match(query))
        .order_by(
            desc(func.ts_rank(Task.search_vector, search_query))
        )
        .limit(limit)
        .offset(offset)
    )
    
    results = (await session.execute(results_stmt)).scalars().all()
    return list(results), total
```

**Verify with EXPLAIN (always):**

```
┌─────────────────────────────────────────────────────────────────┐
│          FULL-TEXT SEARCH EXPLAIN RESULTS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Bitmap Heap Scan on tasks                                      │
│    Recheck Cond: (search_vector @@ '''quarter'' & ''report'''   │
│    → Bitmap Index Scan on ix_tasks_search_vector                │
│        Index Cond: (search_vector @@ '''quarter'' & ''report''' │
│  Planning Time: 0.28 ms                                         │
│  Execution Time: 2.14 ms   ← From 4,723ms to 2ms!             │
│                                                                 │
│  That's 2,200x faster.                                          │
│  And it STAYS fast as the table grows.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 The Search Endpoint

```python
# app/routers/search.py

router = APIRouter(prefix="/search", tags=["search"])


class SearchResultItem(BaseModel):
    id: uuid.UUID
    title: str
    description: str | None
    status: str
    # Include a text snippet with highlighted matches
    model_config = ConfigDict(from_attributes=True)


class SearchResponse(BaseModel):
    results: list[SearchResultItem]
    total: int
    query: str


@router.get("", response_model=SearchResponse)
async def search(
    q: str = Query(min_length=1, max_length=200, description="Search query"),
    limit: int = Query(default=20, ge=1, le=100),
    offset: int = Query(default=0, ge=0),
    current_user: User = Depends(get_current_user),
    session: AsyncSession = Depends(get_session),
):
    """Search tasks within the user's organization.
    
    Uses PostgreSQL full-text search with GIN index.
    Results ranked by relevance.
    Automatically scoped to user's tenant (Week 13, Lecture 2).
    """
    results, total = await search_tasks(
        session=session,
        query=q,
        organization_id=current_user.organization_id,
        limit=limit,
        offset=offset,
    )
    
    return SearchResponse(results=results, total=total, query=q)
```

---

## 4.4 When You'd Need a Dedicated Search Engine

```
┌─────────────────────────────────────────────────────────────────┐
│            SEARCH TECHNOLOGY DECISION FRAMEWORK                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    START HERE                                    │
│                        │                                        │
│                        ▼                                        │
│             ┌────────────────────┐                               │
│             │ Do you need search │                               │
│             │ across 1-2 tables? │                               │
│             └────────┬───────────┘                               │
│                │           │                                    │
│               YES          NO                                   │
│                │           │                                    │
│                ▼           ▼                                    │
│    ┌──────────────┐  ┌─────────────────────┐                    │
│    │ < 10M rows?  │  │ Multiple data        │                    │
│    └──────┬───────┘  │ sources, complex     │                    │
│       │       │      │ faceting, fuzzy,     │                    │
│      YES      NO     │ autocomplete,        │                    │
│       │       │      │ geo-spatial?         │                    │
│       ▼       │      └──────────┬──────────┘                    │
│  ┌──────────┐ │                 │                               │
│  │ PG Full- │ │                 ▼                               │
│  │ Text is  │ │       ┌─────────────────┐                       │
│  │ ENOUGH   │ │       │ Elasticsearch/  │                       │
│  │ ✅       │ │       │ Meilisearch/    │                       │
│  └──────────┘ │       │ Typesense       │                       │
│               │       └─────────────────┘                       │
│               ▼                                                 │
│      ┌──────────────┐                                           │
│      │ PG Full-Text │                                           │
│      │ + Partitioned│                                           │
│      │ tables       │                                           │
│      │ (still works │                                           │
│      │  for most    │                                           │
│      │  SaaS apps)  │                                           │
│      └──────────────┘                                           │
│                                                                 │
│  RULE OF THUMB: PostgreSQL full-text search handles 90% of      │
│  SaaS search needs. Don't add Elasticsearch until PG can't      │
│  keep up. More infrastructure = more things to break.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 Export: The Request That Never Returns

**Show the failure:**

```python
# ❌ This WILL time out for any non-trivial data set

@router.get("/export/tasks")
async def export_tasks(
    current_user: User = Depends(get_current_user),
    session: AsyncSession = Depends(get_session),
):
    # Step 1: Fetch all tasks (might be 100,000 rows)
    tasks = await task_repo.get_all_for_org(
        session, current_user.organization_id
    )  # 3 seconds for 100k rows
    
    # Step 2: Build CSV in memory
    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(["ID", "Title", "Status", "Assignee", "Created"])
    for task in tasks:
        writer.writerow([...])
    # 5 seconds for 100k rows, 200MB string in memory
    
    # Step 3: Return the response
    return Response(
        content=output.getvalue(),  # 200MB response body
        media_type="text/csv",
    )
    # Total: 8+ seconds
    # Load balancer / reverse proxy timeout: 30 seconds
    # With 500k rows: 40+ seconds → 504 GATEWAY TIMEOUT
```

**Ask the class:**

> "What are the THREE things wrong with this code? Think back to what you've learned about background tasks (Week 11), object storage (today), and API design (Week 4)."

**Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│           THREE PROBLEMS WITH SYNC EXPORT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. TIME: Building a large export takes minutes,                │
│     but HTTP requests have timeout limits (30-60s).             │
│     → SOLUTION: Background task (Celery, Week 11)               │
│                                                                 │
│  2. MEMORY: Loading 500k rows + building a CSV string           │
│     consumes hundreds of MB of server RAM.                      │
│     One export request could OOM-kill your server.              │
│     → SOLUTION: Stream rows, write to file, upload to S3        │
│                                                                 │
│  3. DELIVERY: Returning a 200MB response body ties up           │
│     a server connection for the entire transfer duration.       │
│     → SOLUTION: Upload to S3, give client a presigned URL       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.6 The Async Export Pipeline

**The correct pattern: request → accept → process → notify → download.**

```
┌─────────────────────────────────────────────────────────────────┐
│                ASYNC EXPORT ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT                 YOUR API        CELERY         S3       │
│  ──────                 ────────        ──────         ──       │
│     │                      │               │           │        │
│  1. │── POST /exports ────▶│               │           │        │
│     │   {format: "csv"}    │ Create job    │           │        │
│     │                      │ record (DB)   │           │        │
│     │                      │ Queue task    │           │        │
│     │◀── 202 Accepted ─────│               │           │        │
│     │   {export_id,        │               │           │        │
│     │    status: "queued"} │               │           │        │
│     │                      │               │           │        │
│     │   (User can close browser — export continues)    │        │
│     │                      │               │           │        │
│     │                      │            2. │ Query DB   │        │
│     │                      │               │ Build CSV  │        │
│     │                      │               │ Stream rows│        │
│     │                      │               │──Upload──▶│        │
│     │                      │               │           │ Store  │
│     │                      │               │◀── OK ────│        │
│     │                      │               │           │        │
│     │                      │◀─ Update DB ──│           │        │
│     │                      │  status =     │           │        │
│     │                      │  "completed"  │           │        │
│     │                      │               │           │        │
│  3. │── GET /exports/      │               │           │        │
│     │   {export_id} ──────▶│               │           │        │
│     │                      │ Check status  │           │        │
│     │◀── 200 ──────────────│               │           │        │
│     │   {status:"completed"│               │           │        │
│     │    download_url:     │               │           │        │
│     │    "https://s3/..."}│               │           │        │
│     │                      │               │           │        │
│  4. │── GET download_url ──────────────────────────▶│         │
│     │◀── 200 [file bytes] ─────────────────────────│         │
│     │                      │               │           │        │
│                                                                 │
│  ALTERNATIVE TO POLLING (step 3):                               │
│  Push notification via WebSocket (Week 12) when export done.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The export job model:**

```python
# app/models/export_job.py

class ExportJob(Base):
    __tablename__ = "export_jobs"
    
    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    
    # What to export
    export_type: Mapped[str] = mapped_column(String(50))  # "tasks", "members", "audit_log"
    format: Mapped[str] = mapped_column(String(10))        # "csv", "json"
    
    # Lifecycle tracking
    status: Mapped[str] = mapped_column(
        String(20), default="queued"
    )  # queued → processing → completed | failed
    
    progress_percent: Mapped[int] = mapped_column(default=0)  # 0-100
    
    # Result (populated on completion)
    storage_key: Mapped[str | None] = mapped_column(String(500))
    file_size_bytes: Mapped[int | None] = mapped_column(BigInteger)
    row_count: Mapped[int | None] = mapped_column()
    error_message: Mapped[str | None] = mapped_column(Text)
    
    # Ownership and tenant isolation
    requested_by: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"))
    organization_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organizations.id"))
    
    created_at: Mapped[datetime] = mapped_column(default=func.now())
    completed_at: Mapped[datetime | None] = mapped_column()
```

**The Celery export task:**

```python
# app/tasks/exports.py

import csv
import io
import json
from app.core.celery_app import celery_app
from app.integrations.storage import get_storage_service_sync
from app.core.database import get_sync_session  # Sync session for Celery


@celery_app.task(bind=True, max_retries=2)
def generate_export_task(
    self,
    export_job_id: str,
    organization_id: str,
    export_type: str,
    format: str,
) -> None:
    """Generate data export in background.
    
    Runs in Celery worker (sync context).
    Uses sync DB session and sync S3 client.
    """
    storage = get_storage_service_sync()
    
    with get_sync_session() as session:
        # Update status: processing
        job = session.get(ExportJob, export_job_id)
        job.status = "processing"
        session.commit()
        
        try:
            # ── Build the export file ──
            if export_type == "tasks":
                rows = fetch_all_tasks_sync(session, organization_id)
            else:
                raise ValueError(f"Unknown export type: {export_type}")
            
            if format == "csv":
                file_bytes, content_type = build_csv(rows)
            elif format == "json":
                file_bytes, content_type = build_json(rows)
            else:
                raise ValueError(f"Unknown format: {format}")
            
            # ── Upload to S3 ──
            storage_key = (
                f"exports/{organization_id}/"
                f"{export_job_id}.{format}"
            )
            storage.upload_bytes(storage_key, file_bytes, content_type)
            
            # ── Update job: completed ──
            job.status = "completed"
            job.storage_key = storage_key
            job.file_size_bytes = len(file_bytes)
            job.row_count = len(rows)
            job.completed_at = func.now()
            session.commit()
            
        except Exception as exc:
            job.status = "failed"
            job.error_message = str(exc)
            session.commit()
            raise self.retry(exc=exc)


def build_csv(rows: list[dict]) -> tuple[bytes, str]:
    """Build CSV from row dicts. Returns (bytes, content_type)."""
    output = io.StringIO()
    if not rows:
        return b"", "text/csv"
    
    writer = csv.DictWriter(output, fieldnames=rows[0].keys())
    writer.writeheader()
    writer.writerows(rows)
    
    return output.getvalue().encode("utf-8"), "text/csv"


def build_json(rows: list[dict]) -> tuple[bytes, str]:
    """Build JSON from row dicts. Returns (bytes, content_type)."""
    # Use default=str for UUID and datetime serialization
    data = json.dumps(rows, default=str, indent=2)
    return data.encode("utf-8"), "application/json"
```

**The endpoints:**

```python
# app/routers/exports.py

router = APIRouter(prefix="/exports", tags=["exports"])


class ExportRequest(BaseModel):
    export_type: str = Field(pattern=r"^(tasks|members|audit_log)$")
    format: str = Field(pattern=r"^(csv|json)$")


class ExportStatusResponse(BaseModel):
    id: uuid.UUID
    status: str
    progress_percent: int
    download_url: str | None = None
    row_count: int | None = None
    error_message: str | None = None
    created_at: datetime


@router.post("", response_model=ExportStatusResponse, status_code=202)
async def request_export(
    body: ExportRequest,
    current_user: User = Depends(get_current_user),
    session: AsyncSession = Depends(get_session),
):
    """Request a data export. Returns immediately with job ID.
    
    HTTP 202 Accepted: "I've received your request and will
    process it, but it's not done yet."
    (Week 4: HTTP status code semantics)
    """
    job = ExportJob(
        export_type=body.export_type,
        format=body.format,
        requested_by=current_user.id,
        organization_id=current_user.organization_id,
    )
    session.add(job)
    await session.commit()
    
    # Queue background task
    generate_export_task.delay(
        export_job_id=str(job.id),
        organization_id=str(current_user.organization_id),
        export_type=body.export_type,
        format=body.format,
    )
    
    return ExportStatusResponse(
        id=job.id,
        status="queued",
        progress_percent=0,
        created_at=job.created_at,
    )


@router.get("/{export_id}", response_model=ExportStatusResponse)
async def get_export_status(
    export_id: uuid.UUID,
    current_user: User = Depends(get_current_user),
    session: AsyncSession = Depends(get_session),
    storage: StorageService = Depends(get_storage_service),
):
    """Poll export status. Returns download URL when complete."""
    job = await export_repo.get_by_id_and_org(
        session, export_id, current_user.organization_id
    )
    if not job:
        raise HTTPException(status_code=404, detail="Export not found")
    
    download_url = None
    if job.status == "completed" and job.storage_key:
        download_url = storage.generate_download_url(
            key=job.storage_key,
            filename=f"{job.export_type}_{job.id}.{job.format}",
        )
    
    return ExportStatusResponse(
        id=job.id,
        status=job.status,
        progress_percent=job.progress_percent,
        download_url=download_url,
        row_count=job.row_count,
        error_message=job.error_message,
        created_at=job.created_at,
    )
```

---

## 4.7 Common Mistakes

```
┌─────────────────────────────────────────────────────────────────┐
│           SEARCH & EXPORT MISTAKES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SEARCH:                                                        │
│  ───────                                                        │
│  ❌ MISTAKE: Forgetting the GIN index                           │
│     Result: Full-text search works but does sequential scan     │
│     Fix: Always create a GIN index on tsvector columns          │
│     Verify with EXPLAIN (Week 7)                                │
│                                                                 │
│  ❌ MISTAKE: Using raw tsquery with user input                  │
│     Input: "hello & | world" → syntax error → 500              │
│     Fix: Use plainto_tsquery() — handles any input safely       │
│                                                                 │
│  ❌ MISTAKE: Not scoping search to tenant                       │
│     Result: User searches across ALL organizations' data        │
│     Fix: Always include organization_id in WHERE clause         │
│                                                                 │
│  EXPORT:                                                        │
│  ───────                                                        │
│  ❌ MISTAKE: Loading all rows into memory at once               │
│     Result: OOM-kill with large datasets                        │
│     Fix: Use streaming/batched queries for very large exports   │
│                                                                 │
│  ❌ MISTAKE: No expiration on export files in S3                │
│     Result: Storage costs grow forever                          │
│     Fix: S3 lifecycle policy OR Celery Beat cleanup job         │
│     Delete export files older than 7 days                       │
│                                                                 │
│  ❌ MISTAKE: No rate limit on export requests                   │
│     Result: User queues 50 exports → overwhelms workers         │
│     Fix: Limit to N active exports per user/org                 │
│     (Check in endpoint before queuing)                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: ADMIN DASHBOARD & AGGREGATION

## 5.1 The Dashboard That Killed the Database

**Recall the naive version from Part 1. Six separate queries, every page load:**

```
┌─────────────────────────────────────────────────────────────────┐
│           THE DASHBOARD DEATH SPIRAL                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Admin loads dashboard:                                         │
│                                                                 │
│  SELECT COUNT(*) FROM tasks WHERE org_id = $1;          ─┐     │
│  SELECT COUNT(*) FROM tasks WHERE status = 'done' ...;   │     │
│  SELECT COUNT(DISTINCT user_id) FROM task_assignments ..;│     │
│  SELECT status, COUNT(*) FROM tasks GROUP BY status;     ├─ 6  │
│  SELECT DATE_TRUNC('day', ...), COUNT(*) ...             │ DB  │
│     GROUP BY 1 ORDER BY 1;                               │ hits│
│  SELECT u.name, COUNT(t.id) FROM users u                 │     │
│     JOIN tasks t ON ... GROUP BY u.name LIMIT 10;       ─┘     │
│                                                                 │
│  Total: ~200ms per dashboard load                               │
│                                                                 │
│  Now multiply:                                                  │
│  • 50 admins across all tenants                                 │
│  • Auto-refresh every 10 seconds (common in dashboards)         │
│  • 50 × 6 queries × 6 per minute = 1,800 aggregation           │
│    queries per minute                                           │
│                                                                 │
│  These are EXPENSIVE queries — GROUP BY, COUNT, JOIN —          │
│  exactly the kind that make PostgreSQL sweat.                   │
│                                                                 │
│  The irony: Dashboard data barely changes minute to minute.     │
│  You're recomputing the SAME numbers 1,800 times.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The fix has two parts:**

1. **Consolidate queries** (one query with CTEs instead of six round-trips)
2. **Cache the result** (Redis, from Week 10)

---

## 5.2 Aggregation Queries with CTEs

> "You learned CTEs (Common Table Expressions) in Week 5 and used raw SQL via `text()` in Week 7. For complex dashboard queries, a well-structured CTE is clearer and faster than six separate ORM calls."

```python
# app/repositories/analytics.py

from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession


async def get_org_dashboard_stats(
    session: AsyncSession,
    organization_id: uuid.UUID,
) -> dict:
    """Single query that computes all dashboard metrics.
    
    Uses CTEs to organize sub-queries.
    PostgreSQL executes this as one plan — much more efficient
    than 6 separate round-trips.
    
    Why raw SQL here and not ORM?
    Complex aggregations with CTEs, date_trunc, json_agg are
    cleaner and more readable in SQL. The ORM adds verbosity
    without adding safety for read-only analytics queries.
    (Week 7, Lecture 2: "Raw SQL when ORM isn't enough")
    """
    query = text("""
        WITH task_counts AS (
            SELECT
                COUNT(*) AS total,
                COUNT(*) FILTER (WHERE status = 'done') AS completed,
                COUNT(*) FILTER (WHERE status = 'in_progress') AS in_progress,
                COUNT(*) FILTER (WHERE status = 'todo') AS todo,
                COUNT(*) FILTER (
                    WHERE created_at >= NOW() - INTERVAL '7 days'
                ) AS created_last_week
            FROM tasks
            WHERE organization_id = :org_id
              AND deleted_at IS NULL
        ),
        active_members AS (
            SELECT COUNT(DISTINCT user_id) AS count
            FROM task_assignments
            WHERE organization_id = :org_id
              AND assigned_at >= NOW() - INTERVAL '7 days'
        ),
        daily_trend AS (
            SELECT
                DATE_TRUNC('day', created_at)::date AS day,
                COUNT(*) AS created,
                COUNT(*) FILTER (WHERE status = 'done') AS completed
            FROM tasks
            WHERE organization_id = :org_id
              AND created_at >= NOW() - INTERVAL '30 days'
              AND deleted_at IS NULL
            GROUP BY DATE_TRUNC('day', created_at)::date
            ORDER BY day
        ),
        top_members AS (
            SELECT
                u.id,
                u.full_name,
                COUNT(t.id) AS tasks_completed
            FROM users u
            JOIN tasks t ON t.assignee_id = u.id
            WHERE t.organization_id = :org_id
              AND t.status = 'done'
              AND t.completed_at >= NOW() - INTERVAL '30 days'
            GROUP BY u.id, u.full_name
            ORDER BY tasks_completed DESC
            LIMIT 5
        )
        SELECT
            (SELECT row_to_json(task_counts) FROM task_counts)
                AS task_summary,
            (SELECT count FROM active_members)
                AS active_member_count,
            (SELECT json_agg(daily_trend) FROM daily_trend)
                AS daily_trend,
            (SELECT json_agg(top_members) FROM top_members)
                AS top_members
    """)
    
    result = await session.execute(query, {"org_id": str(organization_id)})
    row = result.mappings().one()
    
    return {
        "task_summary": row["task_summary"],
        "active_member_count": row["active_member_count"] or 0,
        "daily_trend": row["daily_trend"] or [],
        "top_members": row["top_members"] or [],
    }
```

**Why CTEs instead of six queries:**

```
┌─────────────────────────────────────────────────────────────────┐
│            6 QUERIES vs 1 CTE QUERY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  6 SEPARATE QUERIES:                                            │
│  ───────────────────                                            │
│  App ──[query 1]──▶ DB ──▶ App                                  │
│  App ──[query 2]──▶ DB ──▶ App                                  │
│  App ──[query 3]──▶ DB ──▶ App                                  │
│  App ──[query 4]──▶ DB ──▶ App                                  │
│  App ──[query 5]──▶ DB ──▶ App                                  │
│  App ──[query 6]──▶ DB ──▶ App                                  │
│                                                                 │
│  6 round-trips × ~5ms network latency = 30ms overhead           │
│  6 query plan compilations                                      │
│  Total: ~200ms                                                  │
│                                                                 │
│                                                                 │
│  1 CTE QUERY:                                                   │
│  ────────────                                                   │
│  App ──[one big query]──▶ DB ──▶ App                            │
│                                                                 │
│  1 round-trip = 5ms overhead                                    │
│  1 query plan (PostgreSQL optimizes CTEs together)              │
│  Total: ~80ms                                                   │
│                                                                 │
│  2.5x faster, even BEFORE caching.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Caching Dashboard Data

> "You learned cache-aside pattern in Week 10. Dashboards are the textbook use case: expensive to compute, tolerant of staleness, frequently accessed."

```python
# app/services/dashboard.py

from app.repositories.analytics import get_org_dashboard_stats


class DashboardService:
    """Dashboard data with Redis caching.
    
    Cache strategy: Time-based invalidation (TTL).
    Why not event-based? Dashboard aggregates over thousands
    of rows — invalidating on every task update would defeat
    the purpose. A 5-minute TTL means data is at most
    5 minutes stale. For dashboards, that's fine.
    """
    
    CACHE_TTL = 300  # 5 minutes
    
    def __init__(self, redis, session):
        self.redis = redis
        self.session = session
    
    async def get_dashboard(self, organization_id: uuid.UUID) -> dict:
        cache_key = f"dashboard:{organization_id}"
        
        # 1. Try cache (Week 10: cache-aside)
        cached = await self.redis.get(cache_key)
        if cached:
            return json.loads(cached)
        
        # 2. Cache miss: compute from DB
        stats = await get_org_dashboard_stats(self.session, organization_id)
        
        # 3. Store in cache with TTL
        await self.redis.setex(
            cache_key,
            self.CACHE_TTL,
            json.dumps(stats, default=str),
        )
        
        return stats
```

**Impact:**

```
┌─────────────────────────────────────────────────────────────────┐
│            WITH CACHING: THE MATH                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT CACHE:                                                 │
│  50 admins × 6 refreshes/min = 300 DB queries/min              │
│  Each query: ~80ms of DB CPU time                               │
│  Total DB load: 24,000ms/min = 40% of one DB CPU core          │
│                                                                 │
│  WITH 5-MIN CACHE:                                              │
│  First request per org per 5 min = DB query (cache miss)        │
│  All subsequent requests = Redis GET (sub-millisecond)          │
│                                                                 │
│  If 10 orgs active:                                             │
│  DB queries: 10 orgs × 12 per hour = 120/hour (was 18,000)     │
│  Redis GETs: ~18,000/hour (fast, cheap, what Redis is for)      │
│                                                                 │
│  DB load reduction: 99.3%                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 The Dashboard Endpoint

```python
# app/routers/admin.py

router = APIRouter(prefix="/admin", tags=["admin"])


class TaskSummary(BaseModel):
    total: int
    completed: int
    in_progress: int
    todo: int
    created_last_week: int


class DailyTrendPoint(BaseModel):
    day: date
    created: int
    completed: int


class TopMember(BaseModel):
    id: uuid.UUID
    full_name: str
    tasks_completed: int


class DashboardResponse(BaseModel):
    task_summary: TaskSummary
    active_member_count: int
    daily_trend: list[DailyTrendPoint]
    top_members: list[TopMember]
    cached: bool  # Let the frontend know if data is from cache
    generated_at: datetime


@router.get("/dashboard", response_model=DashboardResponse)
async def get_dashboard(
    current_user: User = Depends(get_current_admin_user),  # Admin only (Week 9 RBAC)
    session: AsyncSession = Depends(get_session),
    redis: Redis = Depends(get_redis),
):
    service = DashboardService(redis=redis, session=session)
    stats = await service.get_dashboard(current_user.organization_id)
    
    return DashboardResponse(
        task_summary=TaskSummary(**stats["task_summary"]),
        active_member_count=stats["active_member_count"],
        daily_trend=[DailyTrendPoint(**d) for d in stats["daily_trend"]],
        top_members=[TopMember(**m) for m in stats["top_members"]],
        cached=True,  # Could track actual cache hit/miss
        generated_at=datetime.utcnow(),
    )
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              INTEGRATION PATTERNS QUICK REFERENCE               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EMAIL:                                                         │
│  ──────                                                         │
│    Endpoint → Celery task → Email provider API (httpx)          │
│    Webhook endpoint receives delivery status                    │
│    Always: idempotency key, signature verification, DB logging  │
│    Never: send email in request handler                         │
│                                                                 │
│  FILES:                                                         │
│  ──────                                                         │
│    DB stores metadata. S3/MinIO stores bytes.                   │
│    Presigned URLs: client uploads/downloads directly to S3      │
│    Flow: request URL → upload to S3 → confirm → download URL    │
│    Never: store files in PostgreSQL or on server filesystem     │
│                                                                 │
│  SEARCH:                                                        │
│  ───────                                                        │
│    PostgreSQL tsvector + GIN index = fast full-text search       │
│    Computed column auto-updates when text fields change          │
│    plainto_tsquery() for safe user input handling               │
│    ts_rank() for relevance ordering                             │
│    Never: LIKE '%query%' at scale                               │
│                                                                 │
│  EXPORT:                                                        │
│  ───────                                                        │
│    POST /exports → 202 Accepted {export_id}                     │
│    Celery task builds file → uploads to S3                      │
│    GET /exports/{id} → poll status → get download URL           │
│    Never: build large exports in request handler                │
│                                                                 │
│  DASHBOARD:                                                     │
│  ──────────                                                     │
│    Single CTE query for all metrics (not 6 round-trips)         │
│    Cache in Redis with TTL (5 min typical)                      │
│    Never: recompute aggregations on every page load             │
│                                                                 │
│  COMMON THREAD:                                                 │
│  ──────────────                                                 │
│    Your request handler COORDINATES. It doesn't EXECUTE.        │
│    Heavy work → Celery. Storage → S3. Repeated reads → Redis.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: How All Five Patterns Connect

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  THE INTEGRATION MAP                                            │
│  (How patterns reference each other in your capstone)           │
│                                                                 │
│                    ┌──────────────────┐                          │
│                    │   FastAPI Route   │                          │
│                    │   Handlers        │                          │
│                    └────────┬─────────┘                          │
│                             │                                   │
│           ┌────────┬────────┼────────┬────────┐                 │
│           │        │        │        │        │                 │
│           ▼        ▼        ▼        ▼        ▼                 │
│       ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐            │
│       │Email │ │Files │ │Search│ │Export│ │Dash- │            │
│       │      │ │      │ │      │ │      │ │board │            │
│       └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘            │
│          │        │        │        │        │                 │
│          │        │        │        │        │                 │
│    ┌─────▼──┐  ┌──▼───┐ ┌──▼──┐  ┌─▼──────┐ ┌▼─────┐          │
│    │ Celery │  │  S3  │ │ PG  │  │ Celery │ │Redis │          │
│    │ Task   │  │MinIO │ │ GIN │  │ Task   │ │Cache │          │
│    └───┬────┘  └──────┘ │Index│  └───┬────┘ └──────┘          │
│        │        ▲       └─────┘      │   ▲                    │
│        │        │                    │   │                    │
│        │        └────────────────────┘   │                    │
│        │         Export uploads to S3    │                    │
│        │                                 │                    │
│        ▼                                 │                    │
│    ┌────────┐                    ┌───────┘                    │
│    │SendGrid│                    │ Dashboard query            │
│    │  API   │                    │ hits PostgreSQL on         │
│    └────────┘                    │ cache miss only            │
│                                  ▼                            │
│                           ┌──────────┐                         │
│                           │PostgreSQL│                         │
│                           │  (CTEs)  │                         │
│                           └──────────┘                         │
│                                                                 │
│  All five patterns use tools you already know.                  │
│  The lecture taught you how to COMPOSE them into features.      │
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
│  WEEK 13, LECTURE 4 (Next):                                     │
│  └─ Code Review & Technical Communication                       │
│     You'll review each other's integration implementations.     │
│     Can your teammate understand your email service             │
│     abstraction? Is your presigned URL flow documented?         │
│                                                                 │
│  WEEK 14 (Capstone Completion):                                 │
│  └─ Implement these patterns in YOUR capstone project.          │
│     Every feature from today maps to a capstone requirement:    │
│     • Email → "Background jobs (email notifications)"          │
│     • Files → "File attachment support"                        │
│     • Search → "Search and filtering with pagination"          │
│     • Export → Background job, not listed but expected          │
│     • Dashboard → Implicit in admin/aggregation features       │
│                                                                 │
│  WEEK 15 (Docker & CI/CD):                                      │
│  └─ MinIO goes into your Docker Compose.                        │
│     SendGrid API key goes into secrets management.              │
│     CI tests mock both S3 and email services.                   │
│                                                                 │
│  WEEK 16 (System Design):                                       │
│  └─ "Design a file upload system" is a classic interview        │
│     question. You've BUILT one. You can draw the presigned      │
│     URL architecture from memory.                               │
│     "Design a notification system" — same advantage.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```