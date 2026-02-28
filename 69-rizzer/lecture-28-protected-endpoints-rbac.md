# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  VULNERABILITY FIRST, SOLUTION SECOND                           │
│  ────────────────────────────────────                           │
│  Students must SEE the danger of unprotected endpoints          │
│  before writing a single line of auth middleware.               │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Auth is layered. We use a building security analogy            │
│  throughout. Each concept maps to a physical checkpoint.        │
│                                                                 │
│  DEPENDENCY CHAINS ARE THE INSIGHT                              │
│  ─────────────────────────────────                              │
│  The entire auth system is built on Depends() from Week 3.      │
│  OAuth2PasswordBearer is a Depends(). get_current_user is a     │
│  Depends(). Role checking is a Depends(). It's Depends()        │
│  all the way down. Once students see this, auth demystifies.    │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Depends()        → Week 3 Lecture 4 (the foundation)           │
│  JWT decode       → Week 9 Lecture 2 (token verification)       │
│  User model in DB → Week 9 Lecture 1 + Week 6 (SQLAlchemy)      │
│  Custom exceptions → Week 1 Lecture 2 (exception hierarchies)   │
│  Decorators/closures → Week 1 Lecture 2 (dependency factories)  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                 PROTECTED ENDPOINTS & RBAC                      │
│                     (3–4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE UNGUARDED DOOR (30 min)                            │
│  ├─ 1.1 The Vulnerability (Live Demonstration)                  │
│  ├─ 1.2 Three Questions Every Request Must Answer               │
│  ├─ 1.3 The Building Security Analogy                           │
│  └─ 1.4 401 vs 403 — Two Different Rejections                   │
│                                                                 │
│  PART 2: THE IDENTITY CHAIN (50 min)                            │
│  ├─ 2.1 OAuth2PasswordBearer — The Badge Reader                 │
│  ├─ 2.2 The Authorization Header                                │
│  ├─ 2.3 get_current_user — The Security Desk                    │
│  ├─ 2.4 The Full Dependency Chain (Visualized)                  │
│  └─ 2.5 Your First Protected Endpoint                           │
│                                                                 │
│  PART 3: ROLE-BASED ACCESS CONTROL (50 min)                     │
│  ├─ 3.1 Why Identity Alone Isn't Enough                         │
│  ├─ 3.2 Modeling Roles in the Database                          │
│  ├─ 3.3 The Role-Checking Dependency (A Dependency Factory)     │
│  ├─ 3.4 Admin-Only Endpoints                                    │
│  └─ 3.5 Multi-Role Access Patterns                              │
│                                                                 │
│  PART 4: FINE-GRAINED CONTROL WITH SCOPES (40 min)             │
│  ├─ 4.1 When Roles Aren't Granular Enough                       │
│  ├─ 4.2 What Are OAuth2 Scopes?                                 │
│  ├─ 4.3 Defining Scopes in FastAPI                              │
│  ├─ 4.4 Embedding Scopes in Tokens                              │
│  └─ 4.5 Checking Scopes in get_current_user                     │
│                                                                 │
│  PART 5: REAL-WORLD PATTERNS (30 min)                           │
│  ├─ 5.1 Optional Authentication (The Public Lobby)              │
│  ├─ 5.2 Resource Ownership (Your Office, Not Theirs)            │
│  └─ 5.3 Common Mistakes and Misconceptions                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE UNGUARDED DOOR

## 1.1 The Vulnerability

**Start with a demonstration. Make them see the danger.**

This is the Task Manager API they've been building since Week 3. Database-backed since Week 6. The endpoints work. The tests pass. Ship it?

```python
# routes/tasks.py — The endpoint you built last month. Looks fine, right?

@router.delete("/tasks/{task_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_task(
    task_id: UUID,
    session: AsyncSession = Depends(get_session),
):
    task = await task_repo.get_by_id(session, task_id)
    if not task:
        raise HTTPException(status_code=404, detail="Task not found")
    await task_repo.delete(session, task_id)
```

**Now, open a terminal. Play the attacker.**

```bash
# I'm a stranger on the internet. I've never registered. I have no account.
# I guessed a task UUID (or scraped it from a listing endpoint).

curl -X DELETE http://your-api.com/tasks/550e8400-e29b-41d4-a716-446655440000

# Response: 204 No Content
# The task is gone. Forever.
```

**Pause. Let it sink in.**

```bash
# Now I'll delete ALL of Alice's tasks. One by one.
curl -X DELETE http://your-api.com/tasks/550e8400-e29b-41d4-a716-446655440001
curl -X DELETE http://your-api.com/tasks/550e8400-e29b-41d4-a716-446655440002
curl -X DELETE http://your-api.com/tasks/550e8400-e29b-41d4-a716-446655440003

# Or update someone's task to say something else:
curl -X PATCH http://your-api.com/tasks/550e8400-e29b-41d4-a716-446655440004 \
     -H "Content-Type: application/json" \
     -d '{"title": "HACKED", "description": "Your data is mine"}'

# Response: 200 OK
```

**Now ask the class:**

> "This endpoint has perfect Pydantic validation. It has proper status codes. It has test coverage. It even uses the repository pattern. But it has a FATAL flaw. What is it?"

Answer: **It has no idea WHO is making the request. It accepts commands from anyone.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE PROBLEM                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your API right now:                                            │
│                                                                 │
│       Anyone on the internet                                    │
│              │                                                  │
│              │  DELETE /tasks/abc-123                            │
│              │                                                  │
│              ▼                                                  │
│       ┌──────────────┐                                          │
│       │   Your API   │   "Sure! Deleted!"                       │
│       │   (no idea   │                                          │
│       │    who you   │   No questions asked.                    │
│       │    are)      │   No identity check.                     │
│       └──────────────┘   No permission check.                   │
│                                                                 │
│  This is like a bank vault with no lock on the door.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.2 Three Questions Every Request Must Answer

**Every protected endpoint must answer three questions, in order:**

```
┌─────────────────────────────────────────────────────────────────┐
│             THREE QUESTIONS OF EVERY REQUEST                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  QUESTION 1: "ARE YOU ANYONE?"          ← Authentication        │
│  ──────────────────────────────                                 │
│  Does this request carry a valid identity?                      │
│  Is there a token? Is it real? Is it expired?                   │
│                                                                 │
│  If NO → 401 Unauthorized. Stop here.                           │
│                                                                 │
│                                                                 │
│  QUESTION 2: "ARE YOU ALLOWED?"         ← Authorization         │
│  ──────────────────────────────                                 │
│  Does this person have the right role or permissions            │
│  to perform this specific action?                               │
│                                                                 │
│  If NO → 403 Forbidden. Stop here.                              │
│                                                                 │
│                                                                 │
│  QUESTION 3: "IS THIS YOURS?"           ← Ownership             │
│  ────────────────────────────                                   │
│  Even if they're allowed to delete tasks in general,            │
│  are they deleting THEIR OWN task or someone else's?            │
│                                                                 │
│  If NOT THEIRS → 403 Forbidden. Stop here.                      │
│                                                                 │
│                                                                 │
│  Only after all three: ✅ Process the request.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "You already know the difference between authentication and authorization from Lecture 1 this week. Today we BUILD the machinery that enforces both — on every request, automatically."

---

## 1.3 The Building Security Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE BUILDING SECURITY MODEL                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Imagine a corporate office building.                           │
│                                                                 │
│  FLOOR 1 — PUBLIC LOBBY                                         │
│  Anyone can walk in. Browse the company directory.              │
│  No badge needed.                                               │
│                                                                 │
│  FRONT DOOR — BADGE READER                                      │
│  To go past the lobby, you swipe your badge.                    │
│  The reader doesn't know who you are. It just reads             │
│  the badge number and passes it to the security desk.           │
│                                                                 │
│  SECURITY DESK — IDENTITY VERIFICATION                          │
│  Takes the badge number, looks you up in the system.            │
│  "Badge #4821 → Alice Chen, Engineering, Floor 3."              │
│  If the badge is fake or expired: "ACCESS DENIED."              │
│                                                                 │
│  ELEVATOR — FLOOR ACCESS                                        │
│  Alice has clearance for floors 1-3.                            │
│  She presses Floor 7 (Executive). "ACCESS DENIED."              │
│  Only C-suite roles can reach Floor 7.                          │
│                                                                 │
│  ROOM KEYCARDS — SPECIFIC ROOM ACCESS                           │
│  Even on Floor 3, not every room is open.                       │
│  Alice can enter the dev lab but not the server room.           │
│  Each keycard has specific room permissions.                    │
│                                                                 │
│  PERSONAL OFFICE — YOUR STUFF                                   │
│  Alice can enter her own office.                                │
│  She can NOT enter Bob's office, even on the same floor.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to code — reference this throughout the lecture:**

```
Building                    │  Your API
────────────────────────────│──────────────────────────
Security badge              │  JWT token (Lecture 2)
Badge reader at the door    │  OAuth2PasswordBearer
Security desk               │  get_current_user dependency
Floor access levels         │  Role-based access (RBAC)
Room-specific keycards      │  OAuth2 scopes
Public lobby                │  Optional auth endpoints
Personal office             │  Resource ownership
Fake / expired badge        │  Invalid / expired token → 401
Valid badge, wrong floor    │  Valid user, wrong role → 403
```

---

## 1.4 401 vs 403 — Two Different Rejections

**These two status codes are confused constantly, even by experienced developers.**

> "You learned HTTP status codes in Week 3, Lecture 1. Now we see where 401 and 403 actually diverge."

```
┌─────────────────────────────────────────────────────────────────┐
│                    401 VS 403                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  401 UNAUTHORIZED                                               │
│  ─────────────────                                              │
│  "I don't know who you are."                                    │
│                                                                 │
│  When:                                                          │
│  ├─ No token provided at all                                    │
│  ├─ Token is malformed / can't be decoded                       │
│  ├─ Token has expired                                           │
│  ├─ Token's signature doesn't match (tampered)                  │
│  └─ User in token no longer exists in database                  │
│                                                                 │
│  Building: You have no badge, or your badge is fake.            │
│  The door won't even open. Go away.                             │
│                                                                 │
│  HTTP spec REQUIRES a WWW-Authenticate header in the response.  │
│                                                                 │
│                                                                 │
│  403 FORBIDDEN                                                  │
│  ──────────────                                                 │
│  "I know who you are. You're not allowed."                      │
│                                                                 │
│  When:                                                          │
│  ├─ User is authenticated but has the wrong role                │
│  ├─ User's token doesn't have the required scope                │
│  ├─ User is trying to access another user's resource            │
│  └─ User's account is deactivated / suspended                   │
│                                                                 │
│  Building: Your badge works, I see you're Alice from            │
│  Engineering. But this is the Executive floor. Sorry.           │
│                                                                 │
│  No WWW-Authenticate header needed — the problem isn't          │
│  authentication, it's permission.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visualize the checkpoint sequence:**

```
  Request arrives
       │
       ▼
  ┌─────────────┐     ┌──────────────────────────────────┐
  │ Has valid    │ NO  │ 401 Unauthorized                 │
  │ token?       │────▶│ {"detail": "Not authenticated"}  │
  └──────┬───────┘     │ WWW-Authenticate: Bearer         │
         │ YES         └──────────────────────────────────┘
         ▼
  ┌──────────────┐     ┌──────────────────────────────────┐
  │ Has required │ NO  │ 403 Forbidden                    │
  │ role/scope?  │────▶│ {"detail": "Insufficient perms"} │
  └──────┬───────┘     └──────────────────────────────────┘
         │ YES
         ▼
  ┌──────────────┐     ┌──────────────────────────────────┐
  │ Owns this    │ NO  │ 403 Forbidden                    │
  │ resource?    │────▶│ {"detail": "Not your resource"}  │
  └──────┬───────┘     └──────────────────────────────────┘
         │ YES
         ▼
    ✅ Process request
```

**Now ask the class:**

> "If I send a request with NO token, what status code? (401.) If I send Alice's valid token to delete Bob's task, what status code? (403.) If I send a token that expired 5 minutes ago? (401 — the identity can't be verified.) These are different failures at different checkpoints."

---

# PART 2: THE IDENTITY CHAIN

## 2.1 OAuth2PasswordBearer — The Badge Reader

> "Remember Depends() from Week 3, Lecture 4? You used it for database sessions, custom logic, even yield dependencies. The entire auth system in FastAPI is built on the same mechanism. OAuth2PasswordBearer is just a fancy Depends()."

**What OAuth2PasswordBearer actually does:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  OAuth2PasswordBearer                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  It does ONE job:                                               │
│                                                                 │
│  Extract the token string from the Authorization header.        │
│                                                                 │
│  That's it. It does NOT:                                        │
│  ├─ ❌ Verify the token                                         │
│  ├─ ❌ Decode the JWT                                           │
│  ├─ ❌ Look up the user in the database                         │
│  └─ ❌ Check any permissions                                    │
│                                                                 │
│  It is the BADGE READER, not the security desk.                 │
│  It reads what's on the badge. Someone else verifies it.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Creating the badge reader:**

```python
from fastapi.security import OAuth2PasswordBearer

# Create the "badge reader"
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="auth/login")
```

**What is `tokenUrl`?**

```
┌─────────────────────────────────────────────────────────────────┐
│                    tokenUrl EXPLAINED                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  tokenUrl="auth/login"                                          │
│                                                                 │
│  This does NOT create a login endpoint.                         │
│  You already built the login endpoint in Lecture 1–2.           │
│                                                                 │
│  tokenUrl is METADATA for the OpenAPI docs (Swagger UI).        │
│  It tells the Swagger "Authorize" button where to send          │
│  the username/password when a developer wants to test.          │
│                                                                 │
│  ┌──────────────────────────────────────┐                       │
│  │  Swagger UI                          │                       │
│  │  ┌────────────────────┐              │                       │
│  │  │  🔓 Authorize      │              │                       │
│  │  │                    │              │                       │
│  │  │  Username: [alice] │              │                       │
│  │  │  Password: [*****] │              │                       │
│  │  │                    │              │                       │
│  │  │  [Login] → POST /auth/login       │  ← tokenUrl          │
│  │  └────────────────────┘              │                       │
│  └──────────────────────────────────────┘                       │
│                                                                 │
│  In production, your frontend or mobile app sends the           │
│  token directly. tokenUrl only matters for docs.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Using it as a dependency — just like Depends():**

```python
from fastapi import Depends

@router.get("/tasks")
async def list_tasks(
    token: str = Depends(oauth2_scheme),  # ← That's it
):
    # token is now the raw JWT string from the header
    # e.g. "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOi..."
    print(token)
    ...
```

**What happens under the hood when a request arrives:**

```
┌─────────────────────────────────────────────────────────────────┐
│          OAuth2PasswordBearer — UNDER THE HOOD                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Incoming request                                            │
│     Headers:                                                    │
│       Authorization: Bearer eyJhbGciOi...                       │
│                       ▲       ▲                                 │
│                       │       │                                 │
│                    scheme    token                               │
│                                                                 │
│  2. OAuth2PasswordBearer checks:                                │
│     ├─ Is there an Authorization header?                        │
│     ├─ Does it start with "Bearer "?                            │
│     └─ Extract everything after "Bearer "                       │
│                                                                 │
│  3. IF header present and valid format:                         │
│     → Returns the token string to your dependency               │
│                                                                 │
│  4. IF header missing or wrong format:                          │
│     → Raises HTTPException(401) automatically                   │
│     → Response: {"detail": "Not authenticated"}                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 The Authorization Header

**How clients send the token with every request:**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE AUTHORIZATION HEADER                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP Request:                                                  │
│  ──────────────                                                 │
│  GET /tasks HTTP/1.1                                            │
│  Host: api.example.com                                          │
│  Authorization: Bearer eyJhbGciOiJIUzI1NiIs...                  │
│                 ^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^                 │
│                 scheme         token                             │
│                                                                 │
│                                                                 │
│  The flow (from the client's perspective):                      │
│                                                                 │
│  Step 1: Client logs in                                         │
│     POST /auth/login  →  {"access_token": "eyJ...", ...}        │
│                                                                 │
│  Step 2: Client stores the token (memory, cookie, etc.)         │
│     (You covered client-side storage in Lecture 2)              │
│                                                                 │
│  Step 3: Client sends token with EVERY subsequent request       │
│     GET /tasks                                                  │
│     Authorization: Bearer eyJ...                                │
│                                                                 │
│  Step 4: Server extracts and verifies the token                 │
│     (This is what we're building NOW)                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Testing with curl and httpx:**

```bash
# With curl:
curl -H "Authorization: Bearer eyJhbGciOi..." http://localhost:8000/tasks

# Without the header:
curl http://localhost:8000/tasks
# Response: 401 {"detail": "Not authenticated"}
```

```python
# With httpx (you know this from Week 8):
import httpx

async with httpx.AsyncClient() as client:
    response = await client.get(
        "http://localhost:8000/tasks",
        headers={"Authorization": f"Bearer {access_token}"}
    )
```

---

## 2.3 get_current_user — The Security Desk

**The badge reader extracted the token. Now we need to VERIFY it and figure out WHO this person is.**

> "This is where your JWT knowledge from Lecture 2 meets your Depends() knowledge from Week 3. We're chaining dependencies."

**The security desk does three things:**

```
┌─────────────────────────────────────────────────────────────────┐
│               get_current_user — THREE JOBS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JOB 1: DECODE THE TOKEN                                        │
│  ─────────────────────────                                      │
│  Take the raw JWT string, decode it, verify the signature.      │
│  If it's expired or tampered → reject immediately (401).        │
│  (Using decode_access_token from Lecture 2)                     │
│                                                                 │
│  JOB 2: EXTRACT THE USER IDENTITY                               │
│  ────────────────────────────────                               │
│  The JWT payload contains a "sub" (subject) claim.              │
│  That's the user's ID. Pull it out.                             │
│  If there's no "sub" → reject (401).                            │
│                                                                 │
│  JOB 3: LOOK UP THE USER IN THE DATABASE                        │
│  ────────────────────────────────────────                       │
│  Take the user ID, query PostgreSQL, get the full User object.  │
│  If user doesn't exist (deleted since token issued) → 401.      │
│  If user is deactivated → 401.                                  │
│  (Using async SQLAlchemy from Week 6)                           │
│                                                                 │
│  RESULT: A fully verified User object, ready for the route.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**First, a Pydantic model for the data inside the token:**

```python
# schemas/auth.py

from pydantic import BaseModel
from uuid import UUID

class TokenData(BaseModel):
    """The data we extract from a decoded JWT payload"""
    user_id: UUID
```

**Now, build the dependency — step by step:**

```python
# dependencies/auth.py

from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession
from jose import JWTError

from db.session import get_session          # Week 6 — session dependency
from services.auth import decode_access_token  # Lecture 2 — JWT decode
from repositories.user import user_repo     # Week 6 — repository pattern
from models.user import User                # Lecture 1 — SQLAlchemy model
from schemas.auth import TokenData

# Step 1: The badge reader (extracts token string)
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="auth/login")


# Step 2: The security desk (verifies token, returns user)
async def get_current_user(
    token: str = Depends(oauth2_scheme),          # ← Badge reader feeds in
    session: AsyncSession = Depends(get_session),  # ← DB session from Week 6
) -> User:
    """
    The CORE auth dependency.
    
    Takes a raw JWT string → returns a verified User from the database.
    
    This runs on EVERY protected request. It must be:
    - Fast (one DB query at most)
    - Secure (reject anything suspicious)
    - Clear in its error messages (for debugging, not for attackers)
    """
    
    # The error we'll raise for ANY auth failure
    # IMPORTANT: Don't give attackers specific reasons for failure
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},  # Required by HTTP spec
    )
    
    # --- JOB 1: Decode the token ---
    try:
        payload: dict = decode_access_token(token)
    except JWTError:
        # Token is expired, tampered, or malformed
        # We don't tell the client WHICH — that helps attackers
        raise credentials_exception
    
    # --- JOB 2: Extract user identity ---
    user_id_str: str | None = payload.get("sub")
    if user_id_str is None:
        # Token is valid but has no subject — something is wrong
        raise credentials_exception
    
    try:
        token_data = TokenData(user_id=user_id_str)
    except ValueError:
        # "sub" wasn't a valid UUID
        raise credentials_exception
    
    # --- JOB 3: Look up the user ---
    user = await user_repo.get_by_id(session, token_data.user_id)
    
    if user is None:
        # User was deleted after the token was issued
        raise credentials_exception
    
    if not user.is_active:
        # Account deactivated — treat same as not found
        raise credentials_exception
    
    return user
```

**Why one generic error message?**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY NOT SPECIFIC ERROR MESSAGES?                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ DON'T DO THIS:                                              │
│  "Token expired"           → Attacker knows token format valid  │
│  "User not found"          → Attacker knows valid token format  │
│  "Account deactivated"     → Attacker knows user exists         │
│                                                                 │
│  ✅ DO THIS:                                                    │
│  "Could not validate credentials"  → Attacker learns nothing    │
│                                                                 │
│  Every auth failure at the identity layer looks identical        │
│  from the outside. Different failures, same 401 response.       │
│                                                                 │
│  Log the specific reason SERVER-SIDE for debugging:             │
│  logger.warning(f"Auth failed: expired token for {user_id}")    │
│  (You set up structlog in Week 3, Lecture 4)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 The Full Dependency Chain (Visualized)

**This is the critical insight. Look at the chain:**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE DEPENDENCY CHAIN                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP Request                                                   │
│  Authorization: Bearer eyJ...                                   │
│       │                                                         │
│       │  ① OAuth2PasswordBearer extracts token string           │
│       ▼                                                         │
│  ┌──────────────────┐                                           │
│  │ oauth2_scheme    │  Depends()                                │
│  │ Returns: str     │  "eyJhbGciOi..."                          │
│  └────────┬─────────┘                                           │
│           │                                                     │
│           │  ② get_current_user receives token + DB session     │
│           ▼                                                     │
│  ┌──────────────────┐  ┌──────────────────┐                     │
│  │ get_current_user │←─│ get_session       │  Depends()         │
│  │ Decodes JWT      │  │ Returns: Session  │                    │
│  │ Queries DB       │  └──────────────────┘                     │
│  │ Returns: User    │                                           │
│  └────────┬─────────┘                                           │
│           │                                                     │
│           │  ③ Route handler receives verified User             │
│           ▼                                                     │
│  ┌──────────────────┐                                           │
│  │ Route Handler    │                                           │
│  │ async def        │                                           │
│  │   list_tasks(    │                                           │
│  │   user: User     │  ← Fully verified, straight from DB      │
│  │ )                │                                           │
│  └──────────────────┘                                           │
│                                                                 │
│  EVERY step is just Depends(). Nothing magical.                 │
│  FastAPI resolves the chain automatically, just like it         │
│  resolved your database session in Week 6.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "See it? `oauth2_scheme` is a Depends(). `get_session` is a Depends(). `get_current_user` is a Depends() that RECEIVES the output of two other Depends(). It's dependency injection all the way down. Same concept from Week 3, new application."

---

## 2.5 Your First Protected Endpoint

**Now lock that door:**

```python
# routes/tasks.py

from dependencies.auth import get_current_user
from models.user import User

@router.get("/tasks", response_model=list[TaskResponse])
async def list_tasks(
    current_user: User = Depends(get_current_user),  # ← THE LOCK
    session: AsyncSession = Depends(get_session),
):
    """List tasks belonging to the current user"""
    # current_user is now a verified User object from the database.
    # We KNOW who this person is. We can filter by their ID.
    tasks = await task_repo.get_by_owner(session, owner_id=current_user.id)
    return tasks


@router.delete("/tasks/{task_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_task(
    task_id: UUID,
    current_user: User = Depends(get_current_user),  # ← THE LOCK
    session: AsyncSession = Depends(get_session),
):
    """Delete a task — but only if it belongs to the current user"""
    task = await task_repo.get_by_id(session, task_id)
    if not task:
        raise HTTPException(status_code=404, detail="Task not found")
    if task.owner_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not your task")
    await task_repo.delete(session, task_id)
```

**Before and after:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    BEFORE AND AFTER                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE (unprotected):                                          │
│                                                                 │
│      curl -X DELETE /tasks/abc-123                              │
│      → 204 No Content. Task gone. Anyone can do this.           │
│                                                                 │
│                                                                 │
│  AFTER (protected):                                             │
│                                                                 │
│      curl -X DELETE /tasks/abc-123                              │
│      → 401 {"detail": "Not authenticated"}                      │
│         (No token provided)                                     │
│                                                                 │
│      curl -X DELETE /tasks/abc-123 \                            │
│           -H "Authorization: Bearer <bob_token>"                │
│      → 403 {"detail": "Not your task"}                          │
│         (Bob can't delete Alice's task)                         │
│                                                                 │
│      curl -X DELETE /tasks/abc-123 \                            │
│           -H "Authorization: Bearer <alice_token>"              │
│      → 204 No Content                                           │
│         (Alice deleting her own task. Allowed.)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**That single `Depends(get_current_user)` parameter transformed a public endpoint into a protected one.** No decorators wrapping your function. No middleware to configure. Just a dependency parameter. FastAPI handles the rest.

---

# PART 3: ROLE-BASED ACCESS CONTROL

## 3.1 Why Identity Alone Isn't Enough

**Now ask the class:**

> "We can identify who's making the request. Alice is Alice, Bob is Bob. But here's a question: should Alice be able to delete Bob's account? Should every authenticated user be able to see every other user's email? Should any logged-in person be able to change system configuration?"

Answer: **No. Different users have different levels of trust.**

```
┌─────────────────────────────────────────────────────────────────┐
│              IDENTITY ≠ PERMISSION                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Authentication alone (what we built in Part 2):                │
│                                                                 │
│  ┌─────────────┐                                                │
│  │ Logged in?  │─── YES ───▶ Full access to everything          │
│  └─────────────┘                                                │
│                                                                 │
│  That's binary: either you're in or you're out.                 │
│  Like a building where every badge opens every door.            │
│                                                                 │
│                                                                 │
│  Authentication + Authorization (what we need):                 │
│                                                                 │
│  ┌─────────────┐         ┌──────────────┐                       │
│  │ Logged in?  │── YES ──│ What is your │                       │
│  └─────────────┘         │ ROLE?        │                       │
│                          └──────┬───────┘                       │
│                        ┌────────┼────────┐                      │
│                        ▼        ▼        ▼                      │
│                     ADMIN    MEMBER    VIEWER                   │
│                     ┌────┐   ┌────┐   ┌────┐                    │
│                     │CRUD│   │CRU │   │ R  │                    │
│                     │+Mgmt│  │own │   │only│                    │
│                     └────┘   └────┘   └────┘                    │
│                                                                 │
│  Different badges open different doors.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Modeling Roles in the Database

**First, define the roles as a Python Enum:**

> "Enum is a built-in Python type for fixed sets of named values. It prevents typos — you can't accidentally write `'admn'` when the Enum enforces `UserRole.ADMIN`."

```python
# models/enums.py

from enum import Enum

class UserRole(str, Enum):
    """
    User roles for the application.
    
    Inherits from str so the values are plain strings.
    This makes them database-friendly (stored as "admin", "member", etc.)
    and JSON-serializable (Pydantic handles them automatically).
    """
    ADMIN = "admin"
    MEMBER = "member"
    VIEWER = "viewer"
```

**Why `(str, Enum)`?**

```
┌─────────────────────────────────────────────────────────────────┐
│                   WHY str + Enum?                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  class UserRole(Enum):        # Just Enum                       │
│      ADMIN = "admin"                                            │
│                                                                 │
│  UserRole.ADMIN == "admin"    → False 😱                        │
│  The Enum value and a plain string are different types.         │
│                                                                 │
│                                                                 │
│  class UserRole(str, Enum):   # str + Enum                      │
│      ADMIN = "admin"                                            │
│                                                                 │
│  UserRole.ADMIN == "admin"    → True ✅                         │
│  Comparisons with plain strings work.                           │
│  JSON serialization works (Pydantic outputs "admin").           │
│  Database storage works (SQLAlchemy stores "admin").            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Add the role column to your User model:**

> "You built this model in Lecture 1 this week. We're adding one column."

```python
# models/user.py

from sqlalchemy.orm import Mapped, mapped_column
from sqlalchemy import String, Boolean
from models.enums import UserRole

class User(Base):
    __tablename__ = "users"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    email: Mapped[str] = mapped_column(String, unique=True, index=True)
    hashed_password: Mapped[str] = mapped_column(String)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    
    # NEW — the role column
    role: Mapped[str] = mapped_column(
        String, 
        default=UserRole.MEMBER.value,  # New users are members by default
    )
    
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
```

> "Don't forget: adding a column means creating an Alembic migration. You know this from Week 6, Lecture 3."

```bash
alembic revision --autogenerate -m "add_role_to_users"
alembic upgrade head
```

---

## 3.3 The Role-Checking Dependency (A Dependency Factory)

**Now we need a dependency that checks roles. But there's a design decision here.**

We could write a separate function for each role:

```python
# ❌ This works but doesn't scale
async def require_admin(
    current_user: User = Depends(get_current_user),
) -> User:
    if current_user.role != UserRole.ADMIN:
        raise HTTPException(status_code=403, detail="Admin access required")
    return current_user

async def require_member(
    current_user: User = Depends(get_current_user),
) -> User:
    if current_user.role != UserRole.MEMBER:
        raise HTTPException(status_code=403, detail="Member access required")
    return current_user

# ... and another for every role. Copy-paste nightmare.
```

**Better: a dependency factory — a function that CREATES dependencies.**

> "Remember decorators from Week 1, Lecture 2? A decorator is a function that takes a function and returns a new function. A dependency factory is the same idea — a function that takes configuration and returns a new dependency. Same closure pattern, different context."

```python
# dependencies/auth.py

def require_role(*allowed_roles: UserRole):
    """
    Dependency FACTORY — creates a role-checking dependency.
    
    Usage:
        Depends(require_role(UserRole.ADMIN))
        Depends(require_role(UserRole.ADMIN, UserRole.MEMBER))
    
    This is a CLOSURE — the inner function captures allowed_roles 
    from the outer scope. Same pattern as decorators from Week 1.
    """
    
    async def role_checker(
        current_user: User = Depends(get_current_user),  # ← Chains!
    ) -> User:
        if current_user.role not in allowed_roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Insufficient permissions",
            )
        return current_user
    
    return role_checker
```

**Trace the dependency chain now:**

```
┌─────────────────────────────────────────────────────────────────┐
│            THE EXTENDED DEPENDENCY CHAIN                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP Request                                                   │
│  Authorization: Bearer eyJ...                                   │
│       │                                                         │
│       │ ① Extract token                                         │
│       ▼                                                         │
│  oauth2_scheme ──────────────────────────────┐                  │
│       │ token: str                           │                  │
│       │                                      │                  │
│       │ ② Decode token, query DB             │                  │
│       ▼                                      ▼                  │
│  get_current_user ◀──────────────── get_session                 │
│       │ user: User                                              │
│       │                                                         │
│       │ ③ Check: is user.role in allowed_roles?                 │
│       ▼                                                         │
│  role_checker (created by require_role)                          │
│       │ user: User (verified AND authorized)                    │
│       │                                                         │
│       │ ④ Route handler receives fully authorized user          │
│       ▼                                                         │
│  Route Handler                                                  │
│                                                                 │
│  FOUR dependencies deep. FastAPI resolves them all              │
│  automatically, in order, before your route code runs.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Admin-Only Endpoints

**Use the factory to protect admin routes:**

```python
# routes/admin.py

from dependencies.auth import require_role
from models.enums import UserRole

@router.get("/admin/users", response_model=list[UserResponse])
async def list_all_users(
    current_user: User = Depends(require_role(UserRole.ADMIN)),
    session: AsyncSession = Depends(get_session),
):
    """
    List ALL users in the system.
    Only admins can see this — members and viewers get 403.
    """
    users = await user_repo.get_all(session)
    return users


@router.patch("/admin/users/{user_id}/role")
async def change_user_role(
    user_id: UUID,
    role_update: RoleUpdate,  # Pydantic model with new_role field
    current_user: User = Depends(require_role(UserRole.ADMIN)),
    session: AsyncSession = Depends(get_session),
):
    """
    Change another user's role.
    Only admins can promote/demote users.
    """
    if user_id == current_user.id:
        raise HTTPException(
            status_code=400, 
            detail="Cannot change your own role"
        )
    
    target_user = await user_repo.get_by_id(session, user_id)
    if not target_user:
        raise HTTPException(status_code=404, detail="User not found")
    
    target_user.role = role_update.new_role
    await session.commit()
    return {"message": f"Role updated to {role_update.new_role}"}


@router.delete("/admin/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def deactivate_user(
    user_id: UUID,
    current_user: User = Depends(require_role(UserRole.ADMIN)),
    session: AsyncSession = Depends(get_session),
):
    """Deactivate a user account. Admin only."""
    if user_id == current_user.id:
        raise HTTPException(
            status_code=400, 
            detail="Cannot deactivate yourself"
        )
    
    target_user = await user_repo.get_by_id(session, user_id)
    if not target_user:
        raise HTTPException(status_code=404, detail="User not found")
    
    target_user.is_active = False
    await session.commit()
```

**What happens with different users:**

```
┌─────────────────────────────────────────────────────────────────┐
│           ADMIN-ONLY IN ACTION                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Alice (role: admin):                                           │
│     GET /admin/users → 200 OK + list of all users               │
│                                                                 │
│  Bob (role: member):                                            │
│     GET /admin/users → 403 {"detail": "Insufficient perms"}     │
│                                                                 │
│  Charlie (role: viewer):                                        │
│     GET /admin/users → 403 {"detail": "Insufficient perms"}     │
│                                                                 │
│  No token:                                                      │
│     GET /admin/users → 401 {"detail": "Not authenticated"}      │
│                                                                 │
│  Note: Bob and Charlie get 403, not 401.                        │
│  We KNOW who they are. They just don't have the clearance.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Multi-Role Access Patterns

**Most endpoints need more than one role. The factory handles this:**

```python
# endpoints with different access levels

# ADMIN ONLY — system management
@router.delete("/admin/users/{user_id}")
async def deactivate_user(
    current_user: User = Depends(require_role(UserRole.ADMIN)),
    ...
):
    ...

# ADMIN + MEMBER — can create and modify content
@router.post("/tasks", response_model=TaskResponse)
async def create_task(
    task_data: TaskCreate,
    current_user: User = Depends(require_role(UserRole.ADMIN, UserRole.MEMBER)),
    session: AsyncSession = Depends(get_session),
):
    """Both admins and members can create tasks."""
    new_task = Task(
        **task_data.model_dump(),
        owner_id=current_user.id,  # Stamp with owner identity
    )
    session.add(new_task)
    await session.commit()
    await session.refresh(new_task)
    return new_task

# ALL AUTHENTICATED — anyone who's logged in
@router.get("/tasks", response_model=list[TaskResponse])
async def list_own_tasks(
    current_user: User = Depends(get_current_user),  # No role check
    session: AsyncSession = Depends(get_session),
):
    """Any authenticated user can view their own tasks."""
    tasks = await task_repo.get_by_owner(session, owner_id=current_user.id)
    return tasks
```

**The access hierarchy visualized:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   ACCESS HIERARCHY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LAYER        │ DEPENDENCY              │ WHO PASSES            │
│  ─────────────│─────────────────────────│──────────────────     │
│  Public       │ (none)                  │ Everyone              │
│  Authed       │ get_current_user        │ Any valid token       │
│  Member+      │ require_role(M, A)      │ Members & Admins      │
│  Admin only   │ require_role(A)         │ Admins only           │
│               │                         │                       │
│               │                         │                       │
│       ┌───────────────────────────────────────────────┐         │
│       │                 ADMIN                          │         │
│       │    ┌───────────────────────────────────┐      │         │
│       │    │          MEMBER                    │      │         │
│       │    │    ┌───────────────────────┐       │      │         │
│       │    │    │       VIEWER          │       │      │         │
│       │    │    │   ┌─────────────┐     │       │      │         │
│       │    │    │   │  AUTHED     │     │       │      │         │
│       │    │    │   │  (any user) │     │       │      │         │
│       │    │    │   └─────────────┘     │       │      │         │
│       │    │    └───────────────────────┘       │      │         │
│       │    └───────────────────────────────────┘      │         │
│       └───────────────────────────────────────────────┘         │
│                                                                 │
│  Each outer ring can do everything the inner rings can,         │
│  plus more. Admin ⊃ Member ⊃ Viewer ⊃ Authenticated.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A cleaner alternative — minimum role level:**

If your roles form a clear hierarchy (viewer < member < admin), you can build a permission ladder instead of listing individual roles each time:

```python
# A mapping from role to its "power level"
ROLE_HIERARCHY: dict[UserRole, int] = {
    UserRole.VIEWER: 1,
    UserRole.MEMBER: 2,
    UserRole.ADMIN: 3,
}

def require_min_role(minimum_role: UserRole):
    """Require at least this role level or higher."""
    
    async def role_checker(
        current_user: User = Depends(get_current_user),
    ) -> User:
        user_level = ROLE_HIERARCHY.get(UserRole(current_user.role), 0)
        required_level = ROLE_HIERARCHY[minimum_role]
        
        if user_level < required_level:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Insufficient permissions",
            )
        return current_user
    
    return role_checker


# Usage — much cleaner for hierarchical roles:
@router.get("/tasks")
async def list_tasks(
    user: User = Depends(require_min_role(UserRole.VIEWER)),  # Viewer+
): ...

@router.post("/tasks")
async def create_task(
    user: User = Depends(require_min_role(UserRole.MEMBER)),  # Member+
): ...

@router.delete("/admin/users/{user_id}")
async def deactivate_user(
    user: User = Depends(require_min_role(UserRole.ADMIN)),  # Admin only
): ...
```

> "Use `require_role()` when your access patterns are non-hierarchical (e.g., 'only editors and auditors, not admins'). Use `require_min_role()` when your roles form a clear ladder. Pick the one that matches your domain."

---

# PART 4: FINE-GRAINED CONTROL WITH SCOPES

## 4.1 When Roles Aren't Granular Enough

**Now ask the class:**

> "Imagine you have a MEMBER user, Bob. Bob should be able to read tasks and create tasks. But should Bob be able to delete tasks? What about exporting all tasks as CSV? What if you want Bob's mobile app to only READ tasks, while his web session has full access?"

Roles are *user-level*: Bob IS a member. Full stop. Every token Bob gets grants member-level access. But what if different tokens for the same user should have different capabilities?

```
┌─────────────────────────────────────────────────────────────────┐
│                  ROLES VS SCOPES                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ROLES (User-level)                                             │
│  ──────────────────                                             │
│  Attached to the USER in the database.                          │
│  "Alice IS an admin."                                           │
│  Every token Alice gets reflects her admin role.                │
│  Coarse-grained: admin, member, viewer.                         │
│                                                                 │
│  Building analogy: Your employee badge says "Engineering."      │
│  That's your department. It doesn't change per trip.            │
│                                                                 │
│                                                                 │
│  SCOPES (Token-level)                                           │
│  ────────────────────                                           │
│  Attached to the TOKEN at creation time.                        │
│  "This specific token can read tasks and create tasks."         │
│  Different tokens for the SAME user can have different scopes.  │
│  Fine-grained: tasks:read, tasks:write, tasks:delete,           │
│                users:read, admin:manage.                         │
│                                                                 │
│  Building analogy: Your keycard for Floor 3 is programmed       │
│  with specific ROOM access. You might get a visitor keycard     │
│  that opens the meeting room but not the server room,           │
│  even though your regular keycard opens both.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world scenarios where scopes matter:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY SCOPES EXIST                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCENARIO 1: Third-party API access                             │
│  A third-party app wants to read Alice's tasks                  │
│  (like a calendar integration). You issue a token               │
│  with ONLY tasks:read scope. The app cannot modify              │
│  or delete anything, even though Alice is an admin.             │
│                                                                 │
│  SCENARIO 2: Limited-purpose tokens                             │
│  A "password reset" token should ONLY be able to                │
│  reset the password. It should NOT be able to read              │
│  tasks or manage users, even for an admin.                      │
│                                                                 │
│  SCENARIO 3: Mobile vs web clients                              │
│  The mobile app might get a restricted token                    │
│  (tasks:read only), while the web dashboard gets                │
│  full access (tasks:read, tasks:write, tasks:delete).           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 What Are OAuth2 Scopes?

**Scopes are just strings. Permission labels. That's all.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   SCOPES ARE STRINGS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A scope is a string like "tasks:read" or "admin:manage".       │
│  Convention: "resource:action"                                  │
│                                                                 │
│  Common patterns:                                               │
│  ├─ tasks:read         Read tasks                               │
│  ├─ tasks:write        Create and update tasks                  │
│  ├─ tasks:delete       Delete tasks                             │
│  ├─ users:read         View user profiles                       │
│  ├─ users:manage       Create/update/delete users               │
│  └─ admin              Full system access                       │
│                                                                 │
│  A token contains a LIST of scopes:                             │
│  ┌──────────────────────────────────────────────┐               │
│  │  JWT Payload:                                 │               │
│  │  {                                            │               │
│  │    "sub": "alice-uuid-here",                  │               │
│  │    "scopes": ["tasks:read", "tasks:write"],   │               │
│  │    "exp": 1700000000                          │               │
│  │  }                                            │               │
│  └──────────────────────────────────────────────┘               │
│                                                                 │
│  The endpoint says: "I require tasks:write."                    │
│  The middleware checks: "Does this token have tasks:write?"     │
│  Yes → proceed. No → 403.                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Defining Scopes in FastAPI

**Update OAuth2PasswordBearer with scope definitions:**

```python
# dependencies/auth.py

from fastapi.security import OAuth2PasswordBearer

# Define ALL available scopes for your application
oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="auth/login",
    scopes={
        "tasks:read": "Read your tasks",
        "tasks:write": "Create and update tasks",
        "tasks:delete": "Delete tasks",
        "users:read": "View user profiles",
        "users:manage": "Manage user accounts",
        "admin": "Full administrative access",
    },
)
```

**What this changes:**

```
┌─────────────────────────────────────────────────────────────────┐
│         SCOPE DEFINITIONS IN SWAGGER UI                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────┐               │
│  │  Swagger UI — Authorize                       │               │
│  │                                               │               │
│  │  Username: [alice@example.com]                │               │
│  │  Password: [***********]                      │               │
│  │                                               │               │
│  │  Scopes:                                      │               │
│  │  ☑ tasks:read    — Read your tasks            │               │
│  │  ☑ tasks:write   — Create and update tasks    │               │
│  │  ☐ tasks:delete  — Delete tasks               │               │
│  │  ☐ users:read    — View user profiles         │               │
│  │  ☐ users:manage  — Manage user accounts       │               │
│  │  ☐ admin         — Full administrative access │               │
│  │                                               │               │
│  │  [Authorize]                                  │               │
│  └──────────────────────────────────────────────┘               │
│                                                                 │
│  The scope definitions tell Swagger what's available.           │
│  In production, your login endpoint decides what scopes         │
│  to grant based on the user's role.                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Embedding Scopes in Tokens

**When a user logs in, decide which scopes their token gets based on their role:**

> "You built create_access_token in Lecture 2. We're adding a `scopes` field to the payload."

```python
# services/auth.py

# Map roles to the scopes they're allowed
ROLE_SCOPES: dict[UserRole, list[str]] = {
    UserRole.ADMIN: [
        "tasks:read", "tasks:write", "tasks:delete",
        "users:read", "users:manage", "admin",
    ],
    UserRole.MEMBER: [
        "tasks:read", "tasks:write", "tasks:delete",
        "users:read",
    ],
    UserRole.VIEWER: [
        "tasks:read",
        "users:read",
    ],
}


def create_access_token(
    user_id: UUID, 
    role: UserRole,
    expires_delta: timedelta | None = None,
) -> str:
    """
    Create a JWT with the user's ID AND their scopes.
    
    The scopes are determined by the user's role at login time.
    """
    scopes = ROLE_SCOPES.get(role, [])
    
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=30))
    
    payload = {
        "sub": str(user_id),
        "scopes": scopes,          # ← NEW: scopes in the token
        "exp": expire,
    }
    
    token = jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
    return token
```

**What the tokens look like for different roles:**

```
┌─────────────────────────────────────────────────────────────────┐
│             TOKEN CONTENTS BY ROLE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ADMIN token payload:                                           │
│  {                                                              │
│    "sub": "alice-uuid",                                         │
│    "scopes": ["tasks:read", "tasks:write", "tasks:delete",     │
│               "users:read", "users:manage", "admin"],           │
│    "exp": 1700000000                                            │
│  }                                                              │
│                                                                 │
│  MEMBER token payload:                                          │
│  {                                                              │
│    "sub": "bob-uuid",                                           │
│    "scopes": ["tasks:read", "tasks:write", "tasks:delete",     │
│               "users:read"],                                    │
│    "exp": 1700000000                                            │
│  }                                                              │
│                                                                 │
│  VIEWER token payload:                                          │
│  {                                                              │
│    "sub": "charlie-uuid",                                       │
│    "scopes": ["tasks:read", "users:read"],                      │
│    "exp": 1700000000                                            │
│  }                                                              │
│                                                                 │
│  Same mechanism. Different keys on the keycard.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 Checking Scopes in get_current_user

**FastAPI has built-in machinery for scope checking. We need two new imports:**

```python
from fastapi import Security
from fastapi.security import SecurityScopes
```

**The difference: `Security()` vs `Depends()`:**

```
┌─────────────────────────────────────────────────────────────────┐
│              Security() vs Depends()                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Depends(get_current_user)                                      │
│  ─────────────────────────                                      │
│  "Run this dependency. Give me the result."                     │
│  No scope information is passed.                                │
│                                                                 │
│  Security(get_current_user, scopes=["tasks:read"])              │
│  ─────────────────────────────────────────────────              │
│  "Run this dependency. Give me the result.                      │
│   BUT ALSO: this endpoint requires 'tasks:read' scope."        │
│  The required scopes are forwarded to get_current_user          │
│  via a SecurityScopes parameter.                                │
│                                                                 │
│  Security() is a SUPERSET of Depends(). It does everything      │
│  Depends() does, plus scope forwarding.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Upgrade get_current_user to support scopes:**

```python
# dependencies/auth.py — UPGRADED version

from fastapi import Depends, HTTPException, status, Security
from fastapi.security import OAuth2PasswordBearer, SecurityScopes
from jose import JWTError

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="auth/login",
    scopes={
        "tasks:read": "Read your tasks",
        "tasks:write": "Create and update tasks",
        "tasks:delete": "Delete tasks",
        "users:read": "View user profiles",
        "users:manage": "Manage user accounts",
        "admin": "Full administrative access",
    },
)


async def get_current_user(
    security_scopes: SecurityScopes,                   # ← NEW parameter
    token: str = Depends(oauth2_scheme),
    session: AsyncSession = Depends(get_session),
) -> User:
    """
    UPGRADED: Now checks token scopes in addition to identity.
    
    When called via Security(get_current_user, scopes=["tasks:read"]),
    security_scopes.scopes will be ["tasks:read"].
    
    When called via Depends(get_current_user) (no scopes),
    security_scopes.scopes will be [] — scope check is skipped.
    So this is BACKWARD COMPATIBLE with Part 2's usage.
    """
    
    # Build the WWW-Authenticate header value
    # If scopes are required, include them in the header
    if security_scopes.scopes:
        authenticate_value = f'Bearer scope="{security_scopes.scope_str}"'
    else:
        authenticate_value = "Bearer"
    
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": authenticate_value},
    )
    
    # --- Same as before: decode token, extract user ID ---
    try:
        payload = decode_access_token(token)
    except JWTError:
        raise credentials_exception
    
    user_id_str: str | None = payload.get("sub")
    token_scopes: list[str] = payload.get("scopes", [])
    
    if user_id_str is None:
        raise credentials_exception
    
    # --- NEW: Check scopes BEFORE hitting the database ---
    # If the token doesn't have the required scopes, reject early.
    # No point querying the DB for a user whose token can't do this anyway.
    for required_scope in security_scopes.scopes:
        if required_scope not in token_scopes:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,  # 403, not 401!
                detail="Not enough permissions",
                headers={"WWW-Authenticate": authenticate_value},
            )
    
    # --- Same as before: look up user in DB ---
    try:
        token_data = TokenData(user_id=user_id_str)
    except ValueError:
        raise credentials_exception
    
    user = await user_repo.get_by_id(session, token_data.user_id)
    
    if user is None:
        raise credentials_exception
    
    if not user.is_active:
        raise credentials_exception
    
    return user
```

**Notice the key detail:**

```
┌─────────────────────────────────────────────────────────────────┐
│              SCOPE CHECK = 403, NOT 401                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Token decoding fails    → 401 (identity unknown)               │
│  User not found in DB    → 401 (identity unknown)               │
│  Token missing a scope   → 403 (identity known, permission no)  │
│                                                                 │
│  The token decoded successfully. We KNOW who this is.           │
│  But their token wasn't issued with the needed scope.           │
│  That's a PERMISSION failure, not an IDENTITY failure.          │
│                                                                 │
│  Building: Your badge is real and we verified your identity.    │
│  But your keycard doesn't have access to this specific room.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Using scopes on endpoints:**

```python
# routes/tasks.py

from fastapi import Security
from dependencies.auth import get_current_user


@router.get("/tasks", response_model=list[TaskResponse])
async def list_tasks(
    current_user: User = Security(get_current_user, scopes=["tasks:read"]),
    session: AsyncSession = Depends(get_session),
):
    """Requires tasks:read scope"""
    tasks = await task_repo.get_by_owner(session, owner_id=current_user.id)
    return tasks


@router.post("/tasks", response_model=TaskResponse, status_code=201)
async def create_task(
    task_data: TaskCreate,
    current_user: User = Security(get_current_user, scopes=["tasks:write"]),
    session: AsyncSession = Depends(get_session),
):
    """Requires tasks:write scope"""
    new_task = Task(**task_data.model_dump(), owner_id=current_user.id)
    session.add(new_task)
    await session.commit()
    await session.refresh(new_task)
    return new_task


@router.delete("/tasks/{task_id}", status_code=204)
async def delete_task(
    task_id: UUID,
    current_user: User = Security(get_current_user, scopes=["tasks:delete"]),
    session: AsyncSession = Depends(get_session),
):
    """Requires tasks:delete scope"""
    task = await task_repo.get_by_id(session, task_id)
    if not task:
        raise HTTPException(status_code=404, detail="Task not found")
    if task.owner_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not your task")
    await task_repo.delete(session, task_id)
```

**What happens now:**

```
┌─────────────────────────────────────────────────────────────────┐
│             SCOPES IN ACTION                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Alice (admin, scopes: everything):                             │
│     GET  /tasks   → 200 ✅ (has tasks:read)                     │
│     POST /tasks   → 201 ✅ (has tasks:write)                    │
│     DELETE /tasks  → 204 ✅ (has tasks:delete)                   │
│                                                                 │
│  Bob (member, scopes: read + write + delete + users:read):      │
│     GET  /tasks   → 200 ✅ (has tasks:read)                     │
│     POST /tasks   → 201 ✅ (has tasks:write)                    │
│     DELETE /tasks  → 204 ✅ (has tasks:delete)                   │
│                                                                 │
│  Charlie (viewer, scopes: tasks:read + users:read only):        │
│     GET  /tasks   → 200 ✅ (has tasks:read)                     │
│     POST /tasks   → 403 ❌ (missing tasks:write)                │
│     DELETE /tasks  → 403 ❌ (missing tasks:delete)               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Roles vs Scopes — when to use which:**

```
┌─────────────────────────────────────────────────────────────────┐
│            CHOOSING: ROLES VS SCOPES VS BOTH                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE ROLES ALONE when:                                          │
│  ├─ You have a simple permission model                          │
│  ├─ All tokens for a role have the same permissions             │
│  └─ No third-party API access or limited-purpose tokens         │
│                                                                 │
│  USE SCOPES ALONE when:                                         │
│  ├─ You need per-token permission control                       │
│  ├─ Third-party apps need limited access                        │
│  └─ Different clients get different capability levels           │
│                                                                 │
│  USE BOTH when:                                                 │
│  ├─ Roles determine the MAXIMUM scopes a user CAN get          │
│  ├─ Specific tokens may be issued with FEWER scopes             │
│  └─ This is the most flexible and is what we built above        │
│                                                                 │
│  For your Week 9 project: start with roles. Add scopes only     │
│  if your design demands per-token granularity.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: REAL-WORLD PATTERNS

## 5.1 Optional Authentication (The Public Lobby)

**Some endpoints should work for EVERYONE but behave differently for logged-in users.**

Example: A public task board. Anyone can see task titles. But logged-in users also see which tasks are assigned to them, and the response includes a "is_mine" field.

```
┌─────────────────────────────────────────────────────────────────┐
│           OPTIONAL AUTHENTICATION                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Problem:                                                       │
│                                                                 │
│  OAuth2PasswordBearer raises 401 if no token is provided.       │
│  That's correct for protected endpoints.                        │
│  But for public endpoints, we WANT no-token requests to work.   │
│                                                                 │
│  Building analogy:                                              │
│  The lobby is open to everyone. No badge needed.                │
│  But if you DO have a badge, the receptionist greets you        │
│  by name and pulls up your visitor schedule.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key: `auto_error=False`**

```python
# dependencies/auth.py

# A SECOND badge reader that doesn't block people without badges
oauth2_scheme_optional = OAuth2PasswordBearer(
    tokenUrl="auth/login",
    auto_error=False,  # ← Don't raise 401 if no token. Return None instead.
)


async def get_current_user_optional(
    token: str | None = Depends(oauth2_scheme_optional),
    session: AsyncSession = Depends(get_session),
) -> User | None:
    """
    Tries to identify the user. Returns None if:
    - No token provided (anonymous visitor)
    - Token is invalid or expired
    - User not found in DB
    
    NEVER raises 401. Silently returns None instead.
    """
    if token is None:
        return None
    
    try:
        payload = decode_access_token(token)
    except JWTError:
        return None
    
    user_id_str: str | None = payload.get("sub")
    if user_id_str is None:
        return None
    
    try:
        user_id = UUID(user_id_str)
    except ValueError:
        return None
    
    user = await user_repo.get_by_id(session, user_id)
    
    if user is None or not user.is_active:
        return None
    
    return user
```

**Using optional auth on an endpoint:**

```python
@router.get("/tasks/public", response_model=list[PublicTaskResponse])
async def list_public_tasks(
    current_user: User | None = Depends(get_current_user_optional),
    session: AsyncSession = Depends(get_session),
):
    """
    Public task board — no login required.
    
    Anonymous users:  see task titles and status.
    Logged-in users:  also see "is_mine" flag and priority.
    """
    tasks = await task_repo.get_public_tasks(session)
    
    return [
        PublicTaskResponse(
            id=task.id,
            title=task.title,
            status=task.status,
            # These fields are None for anonymous users
            is_mine=task.owner_id == current_user.id if current_user else None,
            priority=task.priority if current_user else None,
        )
        for task in tasks
    ]
```

**The two behaviors:**

```
┌─────────────────────────────────────────────────────────────────┐
│             SAME ENDPOINT, TWO BEHAVIORS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Anonymous request (no token):                                  │
│  GET /tasks/public                                              │
│  →  200 OK                                                      │
│  [                                                              │
│    {"id": "...", "title": "Fix bug", "status": "open",          │
│     "is_mine": null, "priority": null}                          │
│  ]                                                              │
│                                                                 │
│  Authenticated request (with token):                            │
│  GET /tasks/public                                              │
│  Authorization: Bearer eyJ...                                   │
│  →  200 OK                                                      │
│  [                                                              │
│    {"id": "...", "title": "Fix bug", "status": "open",          │
│     "is_mine": true, "priority": "high"}                        │
│  ]                                                              │
│                                                                 │
│  Both get 200. The logged-in user just sees more.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Resource Ownership (Your Office, Not Theirs)

**RBAC tells you WHAT you can do. Ownership tells you WHO you can do it to.**

A member can edit tasks — but only THEIR tasks. Not someone else's. This is the difference between "can you edit tasks in general?" (role check) and "can you edit THIS task specifically?" (ownership check).

```
┌─────────────────────────────────────────────────────────────────┐
│              ROLE CHECK ≠ OWNERSHIP CHECK                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Role check: "Does this user have the MEMBER role?"             │
│  → Happens in the dependency, BEFORE the route code.            │
│                                                                 │
│  Ownership check: "Does this task belong to this user?"         │
│  → Happens IN the route code, AFTER fetching the resource.      │
│  → You need the resource from the DB to check ownership.        │
│                                                                 │
│  Building analogy:                                              │
│  Floor access (role): "Can Alice go to Floor 3?" → Yes.         │
│  Office access (ownership): "Can Alice enter Room 312?"         │
│  → Only if it's HER office.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Pattern: ownership check as a reusable dependency:**

```python
# dependencies/ownership.py

async def get_owned_task(
    task_id: UUID,
    current_user: User = Depends(get_current_user),
    session: AsyncSession = Depends(get_session),
) -> Task:
    """
    Fetch a task AND verify the current user owns it.
    
    Combines two checks:
    1. Does the task exist? (404 if not)
    2. Does the current user own it? (403 if not)
    
    Returns the task object, ready to use in the route.
    Admins can bypass the ownership check.
    """
    task = await task_repo.get_by_id(session, task_id)
    
    if not task:
        raise HTTPException(status_code=404, detail="Task not found")
    
    # Admins can access any task
    if current_user.role == UserRole.ADMIN:
        return task
    
    # Everyone else can only access their own tasks
    if task.owner_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not your task")
    
    return task
```

**Using it — the route becomes beautifully clean:**

```python
@router.patch("/tasks/{task_id}", response_model=TaskResponse)
async def update_task(
    task_update: TaskUpdate,
    task: Task = Depends(get_owned_task),  # Auth + existence + ownership
    session: AsyncSession = Depends(get_session),
):
    """
    Update a task.
    
    By the time we reach this code:
    ✅ User is authenticated (get_current_user ran)
    ✅ Task exists (get_owned_task checked)
    ✅ User owns this task, or is admin (get_owned_task checked)
    
    The route body only has BUSINESS LOGIC. No auth boilerplate.
    """
    for field, value in task_update.model_dump(exclude_unset=True).items():
        setattr(task, field, value)
    
    await session.commit()
    await session.refresh(task)
    return task


@router.delete("/tasks/{task_id}", status_code=204)
async def delete_task(
    task: Task = Depends(get_owned_task),  # Same dependency. Reusable.
    session: AsyncSession = Depends(get_session),
):
    """Delete a task. Ownership already verified by dependency."""
    await session.delete(task)
    await session.commit()
```

**The full chain for an ownership-checked endpoint:**

```
┌─────────────────────────────────────────────────────────────────┐
│         THE COMPLETE AUTH CHAIN FOR OWNED RESOURCES              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PATCH /tasks/abc-123                                           │
│  Authorization: Bearer eyJ...                                   │
│       │                                                         │
│       ▼                                                         │
│  ① oauth2_scheme          → Extracts "eyJ..." from header      │
│       │                                                         │
│       ▼                                                         │
│  ② get_current_user       → Decodes JWT, gets User from DB     │
│       │                      (401 if anything fails)            │
│       ▼                                                         │
│  ③ get_owned_task         → Fetches task abc-123 from DB       │
│       │                      (404 if not found)                 │
│       │                      (403 if not yours)                 │
│       ▼                                                         │
│  ④ update_task()          → Only business logic here            │
│       │                      User is verified.                  │
│       │                      Task exists and is theirs.         │
│       ▼                      Just do the update.                │
│  200 OK + updated task                                          │
│                                                                 │
│  FOUR layers of dependencies. Zero auth code in the route.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Common Mistakes and Misconceptions

### Mistake 1: Checking authorization in every route manually

```python
# ❌ WRONG: Copy-pasting auth checks in every route
@router.delete("/tasks/{task_id}")
async def delete_task(
    task_id: UUID,
    token: str = Depends(oauth2_scheme),
    session: AsyncSession = Depends(get_session),
):
    payload = decode_access_token(token)
    user_id = payload.get("sub")
    user = await user_repo.get_by_id(session, user_id)
    if not user:
        raise HTTPException(status_code=401)
    if user.role != "admin":
        raise HTTPException(status_code=403)
    task = await task_repo.get_by_id(session, task_id)
    if task.owner_id != user.id:
        raise HTTPException(status_code=403)
    # FINALLY, the actual logic...
    await task_repo.delete(session, task_id)

# ✅ CORRECT: Push auth into dependencies
@router.delete("/tasks/{task_id}")
async def delete_task(
    task: Task = Depends(get_owned_task),
    session: AsyncSession = Depends(get_session),
):
    await session.delete(task)
    await session.commit()
```

**The route body should contain ONLY business logic.** Auth belongs in dependencies. That's what they're for. If you're writing `if user.role != ...` inside a route handler, you've put security logic in the wrong layer.

---

### Mistake 2: Using 401 when you mean 403

```python
# ❌ WRONG: User IS authenticated but lacks permission
if current_user.role != UserRole.ADMIN:
    raise HTTPException(status_code=401)  # Wrong! You know who they are.

# ✅ CORRECT: 
if current_user.role != UserRole.ADMIN:
    raise HTTPException(status_code=403)  # Right! Identity is fine, permission isn't.
```

```
401 = "Who are you?"     (identity failure — badge doesn't work)
403 = "You can't do that" (permission failure — badge works, door locked)
```

---

### Mistake 3: Trusting the token for role data without DB check

```python
# ❌ DANGEROUS: Reading role directly from the JWT payload
async def get_current_user(token: str = Depends(oauth2_scheme)):
    payload = decode_access_token(token)
    return {"id": payload["sub"], "role": payload["role"]}
    # What if the user's role was changed to "viewer" 5 minutes ago?
    # This token still says "admin" until it expires!

# ✅ SAFER: Always get the CURRENT role from the database
async def get_current_user(token: str = Depends(oauth2_scheme), ...):
    payload = decode_access_token(token)
    user = await user_repo.get_by_id(session, payload["sub"])
    return user  # user.role is the CURRENT role from the DB
```

> "Scopes in the token are fine for immediate permission checks — that's their purpose. But roles can change. If an admin demotes a user, the token's role claim is stale. Always verify critical role data against the database."

---

### Mistake 4: Forgetting `WWW-Authenticate` header on 401 responses

```python
# ❌ WRONG: Missing the required header
raise HTTPException(status_code=401, detail="Not authenticated")

# ✅ CORRECT: HTTP spec requires this header on 401 responses
raise HTTPException(
    status_code=401,
    detail="Not authenticated",
    headers={"WWW-Authenticate": "Bearer"},  # Required!
)
```

The `WWW-Authenticate` header tells the client which authentication scheme to use. Without it, clients and proxies may not know how to authenticate. The HTTP specification makes this header mandatory for 401 responses.

---

### Mistake 5: Letting admins bypass ownership without intention

```python
# ❌ SUBTLE BUG: Admin deletes their own admin account
@router.delete("/admin/users/{user_id}")
async def deactivate_user(
    user_id: UUID,
    current_user: User = Depends(require_role(UserRole.ADMIN)),
    ...
):
    user = await user_repo.get_by_id(session, user_id)
    user.is_active = False  # What if user_id == current_user.id?
    await session.commit()  # Admin just locked themselves out!

# ✅ CORRECT: Self-action guard
@router.delete("/admin/users/{user_id}")
async def deactivate_user(
    user_id: UUID,
    current_user: User = Depends(require_role(UserRole.ADMIN)),
    ...
):
    if user_id == current_user.id:
        raise HTTPException(status_code=400, detail="Cannot deactivate yourself")
    ...
```

Always guard against self-destructive admin operations: deactivating yourself, changing your own role, deleting your own account. These are logic bugs that auth middleware can't catch because the user technically has permission.

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│            PROTECTED ENDPOINTS QUICK REFERENCE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SETUP THE BADGE READER:                                        │
│      oauth2_scheme = OAuth2PasswordBearer(tokenUrl="auth/login")│
│                                                                 │
│  ANY AUTHENTICATED USER:                                        │
│      current_user: User = Depends(get_current_user)             │
│                                                                 │
│  SPECIFIC ROLE:                                                 │
│      admin: User = Depends(require_role(UserRole.ADMIN))        │
│                                                                 │
│  MULTIPLE ROLES:                                                │
│      user: User = Depends(require_role(UserRole.ADMIN,          │
│                                        UserRole.MEMBER))        │
│                                                                 │
│  MINIMUM ROLE LEVEL:                                            │
│      user: User = Depends(require_min_role(UserRole.MEMBER))    │
│                                                                 │
│  SPECIFIC SCOPE:                                                │
│      user: User = Security(get_current_user,                    │
│                            scopes=["tasks:write"])              │
│                                                                 │
│  OPTIONAL AUTH (public + extra for logged-in):                  │
│      user: User | None = Depends(get_current_user_optional)     │
│                                                                 │
│  OWNED RESOURCE (fetch + verify ownership):                     │
│      task: Task = Depends(get_owned_task)                       │
│                                                                 │
│                                                                 │
│  STATUS CODES:                                                  │
│      401 = "Who are you?"        (no/bad/expired token)         │
│      403 = "You can't do this"   (wrong role/scope/ownership)   │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ Auth logic in route body → Use dependencies              │
│      ❌ 401 for permission failures → Use 403                    │
│      ❌ Trust role from JWT only → Verify against DB             │
│      ❌ Missing WWW-Authenticate → Required on all 401s         │
│      ❌ Admin self-destruction → Guard self-actions              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  AUTH IN FASTAPI = DEPENDENCY CHAINS                            │
│                                                                 │
│  Everything is Depends(). The entire auth system is built       │
│  by composing small, focused dependencies:                      │
│                                                                 │
│  ┌──────────────────┐                                           │
│  │ OAuth2Password   │  "Read the badge"                         │
│  │ Bearer           │  Extracts token string from header        │
│  └────────┬─────────┘                                           │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────┐                                           │
│  │ get_current_user │  "Verify the badge, identify the person"  │
│  │                  │  Decodes JWT, queries DB, returns User    │
│  └────────┬─────────┘                                           │
│           │                                                     │
│       ┌───┴────────────────────┐                                │
│       ▼                        ▼                                │
│  ┌──────────────┐    ┌─────────────────┐                        │
│  │ require_role │    │ Security(scopes)│                        │
│  │              │    │                 │                        │
│  │ "Check floor │    │ "Check keycard  │                        │
│  │  clearance"  │    │  room access"   │                        │
│  └──────┬───────┘    └────────┬────────┘                        │
│         │                     │                                 │
│         └──────────┬──────────┘                                 │
│                    ▼                                            │
│           ┌─────────────────┐                                   │
│           │ get_owned_task  │  "Is this YOUR office?"           │
│           │                 │   Fetches resource, checks owner  │
│           └────────┬────────┘                                   │
│                    │                                            │
│                    ▼                                            │
│           ┌─────────────────┐                                   │
│           │ Route Handler   │  Pure business logic.             │
│           │                 │  No auth code. Clean.             │
│           └─────────────────┘                                   │
│                                                                 │
│  THE BUILDING SECURITY ANALOGY:                                 │
│  ├─ Badge reader      = OAuth2PasswordBearer                    │
│  ├─ Security desk     = get_current_user                        │
│  ├─ Floor clearance   = require_role (RBAC)                     │
│  ├─ Room keycards     = OAuth2 scopes                           │
│  ├─ Personal office   = Resource ownership                      │
│  └─ Public lobby      = Optional auth (auto_error=False)        │
│                                                                 │
│  Each layer is a Depends(). Each adds one check.                │
│  Stack them to build exactly the security policy you need.      │
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
│  WEEK 9, LECTURE 4 (Next lecture): API Security                  │
│  └─ CORS configuration — which origins can call your API        │
│     Rate limiting on login — prevent brute force attacks         │
│     OWASP Top 10 — the checklist of what can go wrong           │
│     You'll build on today's auth to HARDEN it.                  │
│                                                                 │
│  WEEK 9 PROJECT: Add Auth to Task Manager                       │
│  └─ You'll wire EVERYTHING from this lecture into your          │
│     existing Task Manager: JWT auth, RBAC, protected            │
│     resources, user-specific data isolation.                    │
│     Today's code IS the project foundation.                     │
│                                                                 │
│  WEEK 10 (Redis & Caching):                                     │
│  └─ Storing refresh tokens in Redis (fast lookup + TTL)         │
│     Token revocation on logout (delete from Redis)              │
│     Your get_current_user might check Redis before the DB.      │
│                                                                 │
│  WEEK 12 (WebSockets):                                          │
│  └─ WebSocket authentication uses the SAME get_current_user     │
│     but token comes in query param or first message,            │
│     not the Authorization header. Same chain, different input.  │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ Multi-tenant RBAC: users belong to ORGANIZATIONS,           │
│     and roles are PER-ORG. Alice is admin of Org A but          │
│     member of Org B. Same Depends() pattern, more complex       │
│     role checking. The foundation you built today scales.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```