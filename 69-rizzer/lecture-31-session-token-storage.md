# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SECURITY INCIDENT FIRST                                        │
│  ───────────────────────                                        │
│  Students must feel the PANIC of a compromised account with     │
│  no kill switch. Then we hand them the kill switch.             │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Building security badge system. Every concept maps to a        │
│  physical card, a scanner, a registry desk, or a power outage. │
│                                                                 │
│  BRIDGE TWO WORLDS                                              │
│  ────────────────                                               │
│  This lecture is the junction of Week 9 (Auth) and Week 10     │
│  (Redis). Students discover WHY these topics were sequenced.   │
│                                                                 │
│  FAILURE-DRIVEN DESIGN                                          │
│  ─────────────────────                                          │
│  Production systems fail. We teach building for the failure     │
│  case from the start — not as a footnote.                       │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  JWT auth (Week 9)    → tokens we're now storing server-side   │
│  Redis types (L1)     → SET, HASH, SORTED SET in practice     │
│  Caching patterns (L2)→ same Redis client, different purpose   │
│  Circuit breaker (Wk8)→ fallback when Redis is unreachable     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                  SESSION & TOKEN STORAGE                        │
│                     (3.5-4 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (30 min)                                   │
│  ├─ 1.1 The 2 AM Incident (Scenario)                            │
│  ├─ 1.2 The Stateless Paradox                                   │
│  ├─ 1.3 Why Not Just PostgreSQL?                                │
│  └─ 1.4 The Building Security Analogy                           │
│                                                                 │
│  PART 2: REFRESH TOKENS IN REDIS (50 min)                       │
│  ├─ 2.1 Why Redis for Tokens?                                   │
│  ├─ 2.2 Token Key Design                                        │
│  ├─ 2.3 Storing and Retrieving Tokens                           │
│  ├─ 2.4 Token Rotation                                          │
│  └─ 2.5 Multi-Device Token Tracking                             │
│                                                                 │
│  PART 3: TOKEN REVOCATION (45 min)                              │
│  ├─ 3.1 When Revocation Matters                                 │
│  ├─ 3.2 Allowlist vs Blocklist                                  │
│  ├─ 3.3 Single-Session Logout                                   │
│  ├─ 3.4 All-Device Logout                                       │
│  ├─ 3.5 Access Token Blocklisting                               │
│  └─ 3.6 Revocation in Your FastAPI Dependencies                 │
│                                                                 │
│  PART 4: DISTRIBUTED RATE LIMITING (40 min)                     │
│  ├─ 4.1 The Multi-Instance Problem                              │
│  ├─ 4.2 Fixed Window Counter (INCR + EXPIRE)                    │
│  ├─ 4.3 Sliding Window (Sorted Sets)                            │
│  └─ 4.4 Rate Limiter as FastAPI Dependency                      │
│                                                                 │
│  PART 5: SESSION DATA STORAGE (25 min)                          │
│  ├─ 5.1 Sessions vs JWT Claims                                  │
│  ├─ 5.2 Redis Hash for Session Data                             │
│  └─ 5.3 Session Lifecycle                                       │
│                                                                 │
│  PART 6: GRACEFUL DEGRADATION (30 min)                          │
│  ├─ 6.1 When Redis Goes Down                                    │
│  ├─ 6.2 Fallback Strategies by Feature                          │
│  ├─ 6.3 Implementing Degradation                                │
│  └─ 6.4 The Degradation Decision Framework                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The 2 AM Incident

**Start with a scenario. Make them sweat.**

> "It's Saturday, 2 AM. Your phone buzzes. A user emails: *'Someone is in my account downloading my files RIGHT NOW.'* You open your admin dashboard. You can see it — requests from an IP in another country, hitting the API every few seconds. The attacker has a valid session. You need to kill it. What do you do?"

**Let's walk through what happens with the auth system you built in Week 9:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE 2 AM TIMELINE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  2:00 AM — User reports: "Someone is in my account!"            │
│  2:01 AM — You confirm: active requests from unknown IP         │
│                                                                 │
│  2:02 AM — You change the user's password in PostgreSQL         │
│                                                                 │
│  2:02 AM — Attacker's access token JWT is still valid ⚠️        │
│            (it was signed 5 minutes ago, expires in 10 min)     │
│                                                                 │
│  2:02–2:12 AM — Attacker keeps using the access token           │
│                 You watch. Helplessly. 😰                        │
│                                                                 │
│  2:12 AM — Access token finally expires                         │
│                                                                 │
│  2:12 AM — Attacker sends refresh token → gets NEW access JWT   │
│            Password change didn't invalidate the refresh token! │
│                                                                 │
│  2:12 AM — Attacker has a fresh 15-minute access token 🔥       │
│                                                                 │
│  2:12–2:27 AM — Attack continues...                             │
│                                                                 │
│  This cycle repeats until the refresh token expires             │
│  (7 DAYS from now).                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "You changed the password. You updated the database. Why is the attacker STILL in? What went wrong?"

Answer: **Nothing went wrong with your code. This is a design gap. Your JWT tokens are self-contained — the server has no way to reach into an already-issued token and invalidate it.**

---

## 1.2 The Stateless Paradox

**The strength of JWTs is also their weakness.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE STATELESS PARADOX                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  In Week 9, you learned:                                        │
│                                                                 │
│     "JWTs are STATELESS — the server doesn't need to            │
│      store anything. The token carries its own proof."          │
│                                                                 │
│  That's true. And it's a superpower:                            │
│  ├─ No database lookup to verify a request                      │
│  ├─ Any server instance can validate the token                  │
│  ├─ Scales horizontally with zero shared state                  │
│                                                                 │
│  But stateless also means:                                      │
│  ├─ The server has NO MEMORY of which tokens exist              │
│  ├─ There is no "revoke" operation on a signed JWT              │
│  ├─ A valid token stays valid until it expires                  │
│  ├─ Changing passwords, roles, or permissions has NO EFFECT     │
│  │   on already-issued tokens                                   │
│                                                                 │
│  The token is a signed contract. Once signed, the server        │
│  cannot un-sign it. It can only wait for it to expire.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The core tension:**

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  STATELESS (pure JWT)       STATEFUL (server-side tracking)  │
│  ─────────────────────      ──────────────────────────────   │
│  ✅ Fast verification       ✅ Instant revocation             │
│  ✅ No shared state          ✅ Full control over sessions     │
│  ✅ Scales easily            ✅ Audit who is logged in         │
│                                                              │
│  ❌ Cannot revoke            ❌ Requires a shared store        │
│  ❌ Cannot force logout      ❌ Store must be fast             │
│  ❌ Stale claims until       ❌ Store is a new failure point   │
│     expiry                                                   │
│                                                              │
│                                                              │
│  THE SOLUTION: Hybrid. Keep JWT for access tokens            │
│  (fast, short-lived). Add server-side tracking for           │
│  refresh tokens and revocation (control, security).          │
│                                                              │
│  The server-side store? Redis.                               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 1.3 Why Not Just PostgreSQL?

**A fair question. You already have a database. Why not use it?**

> "In Week 9 you stored user credentials in PostgreSQL. You could store refresh tokens there too. Let's see why that becomes painful."

```
┌─────────────────────────────────────────────────────────────────┐
│            TOKEN LOOKUP ON EVERY AUTHENTICATED REQUEST          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Option A: Check token revocation against PostgreSQL            │
│                                                                 │
│  Client ──▶ API Server ──▶ PostgreSQL                           │
│    │                          │                                 │
│    │  "Is this token revoked?" │                                │
│    │                          │  Disk seek, connection pool,    │
│    │                          │  query parse, index lookup...   │
│    │                          │                                 │
│    │   ◀── "No, it's valid" ──┘  ~2-10ms per query              │
│    │                                                            │
│    │  × 1,000 requests/second = 1,000 DB queries/second         │
│    │  JUST for token checks. On top of your actual queries.     │
│                                                                 │
│                                                                 │
│  Option B: Check against Redis                                  │
│                                                                 │
│  Client ──▶ API Server ──▶ Redis                                │
│    │                          │                                 │
│    │  "Is this token revoked?" │                                │
│    │                          │  In-memory lookup, no disk.     │
│    │                          │                                 │
│    │   ◀── "No, it's valid" ──┘  ~0.1-0.5ms per query           │
│    │                                                            │
│    │  Redis was BUILT for this: fast, ephemeral key lookups     │
│    │  with automatic expiration.                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Three reasons Redis wins for token storage:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. SPEED                                                       │
│     Redis: sub-millisecond lookups (in-memory)                  │
│     PostgreSQL: 2-10ms per query (disk + connection overhead)   │
│     Token checks happen on EVERY request — speed matters.       │
│                                                                 │
│  2. BUILT-IN TTL                                                │
│     Redis: SET key value EX 604800  ← expires in 7 days, done  │
│     PostgreSQL: Need a cron job or background task to           │
│                 periodically DELETE expired rows.                │
│     Tokens are TEMPORARY by nature. TTL is the right primitive. │
│                                                                 │
│  3. ATOMIC COUNTERS                                             │
│     Redis: INCR ratelimit:login:user42  ← atomic, one command  │
│     PostgreSQL: SELECT count, then UPDATE, with locking...      │
│     Rate limiting auth endpoints requires fast atomic ops.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**To be clear:**

> "PostgreSQL is still your source of truth for users, roles, and permanent data. Redis is your fast, ephemeral layer for 'who is logged in right now' and 'which tokens are currently alive.' They serve different purposes."

---

## 1.4 The Building Security Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE BUILDING SECURITY ANALOGY                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your application is a SECURE OFFICE BUILDING.                  │
│                                                                 │
│  PURE JWT (Week 9 — no server-side tracking):                   │
│  ─────────────────────────────────────────────                  │
│  You hand every employee a badge with an expiry date            │
│  stamped on it. The badge scanners at the doors check           │
│  the stamp — no network call, no registry. Fast.                │
│                                                                 │
│  But if an employee is fired at 9 AM, their badge               │
│  still opens doors until midnight. You can't deactivate         │
│  it because nothing tracks which badges are "live."             │
│                                                                 │
│                                                                 │
│  JWT + REDIS (this lecture):                                     │
│  ──────────────────────────                                     │
│  Same badges, same scanners. But NOW you also have a            │
│  SECURITY DESK with a live registry of all active badges.       │
│                                                                 │
│  Employee fired at 9 AM? Security desk removes their            │
│  badge number from the registry. Next time they scan,           │
│  the system checks the registry: "Badge #4521? Not on           │
│  the active list. ACCESS DENIED."                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to our system:**

```
Building Security             │  Our Auth System
──────────────────────────────│──────────────────────────────
Employee                      │  User
Badge (key card)              │  Access token (JWT)
Master door code / fob        │  Refresh token
Badge scanner at every door   │  JWT signature check
Expiry date stamped on badge  │  JWT exp claim
Security desk registry        │  Redis
HR database                   │  PostgreSQL
"Deactivate badge #4521"      │  DEL refresh_token:{jti}
"Deactivate ALL badges        │  Delete all keys for user
  for Employee X"             │    (all-device logout)
Badge auto-expires on date    │  Redis TTL
"Max 3 scans/min or lockout"  │  Rate limiting (INCR+EXPIRE)
Employee preference card      │  Session data (Redis Hash)
  (floor access, desk prefs)  │
Power outage at security desk │  Redis goes down
  → scanners still read badge │  → JWT-only validation
  → but can't deactivate      │  → but can't revoke
Multiple office branches      │  Multiple server instances
  → need ONE central registry │  → need ONE shared Redis
```

> "For the rest of this lecture, when I say 'security desk,' think Redis. When I say 'badge,' think token. When I say 'scanner,' think JWT signature check."

---

# PART 2: REFRESH TOKENS IN REDIS

## 2.1 Why Redis for Tokens?

**Quick connection — you already know all the building blocks:**

> "In Lecture 1, you learned Redis data types: Strings, Hashes, Sets, Sorted Sets, and TTL. In Lecture 2, you connected redis.asyncio to FastAPI as a dependency. In Week 9, you built JWT auth with refresh tokens. Now we combine all three."

```
┌─────────────────────────────────────────────────────────────────┐
│             WHAT REDIS GIVES US FOR TOKEN STORAGE               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Redis Feature          │  Token Storage Use                    │
│  ───────────────────────│──────────────────────────────────     │
│  SET key value EX ttl   │  Store token with auto-expiry         │
│  GET key                │  Validate: "does this token exist?"   │
│  DEL key                │  Revoke a single token instantly      │
│  SADD / SMEMBERS        │  Track all tokens for one user        │
│  EXISTS key             │  Check blocklist in O(1)              │
│  INCR + EXPIRE          │  Rate limit auth endpoints            │
│  HSET / HGETALL         │  Store session data as a hash         │
│  Pipeline               │  Atomic multi-step operations         │
│                                                                 │
│  Everything you need was covered in Lectures 1 and 2.           │
│  This lecture is about APPLYING those tools to authentication.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 Token Key Design

**Before writing any code, design your key structure.**

> "In Lecture 2, you learned about cache key namespacing. Token keys follow the same principle, but the stakes are higher — a key collision here means one user could validate using another user's token."

**The JTI (JWT ID) — your token's fingerprint:**

In Week 9, you included standard claims in your JWT payload: `sub` (user ID), `exp` (expiry), `iat` (issued at). There's one more standard claim we now need: `jti` — a unique identifier for each individual token.

```python
import uuid

def create_token_payload(user_id: int) -> dict:
    now = datetime.utcnow()
    return {
        "sub": str(user_id),
        "exp": now + timedelta(days=7),
        "iat": now,
        "jti": str(uuid.uuid4()),  # ← Unique ID for THIS specific token
    }
```

**Why JTI and not the full token as the key?**

```
┌─────────────────────────────────────────────────────────────────┐
│                 TOKEN AS KEY VS JTI AS KEY                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ Full token as Redis key:                                     │
│     Key: "eyJhbGciOiJIUzI1NiIsInR5cCI6Ikp..."  (300+ chars)    │
│     Problems:                                                   │
│     ├─ Wastes memory (long keys × thousands of tokens)          │
│     ├─ Slower lookups (key comparison is O(key_length))         │
│     └─ Token visible in Redis CLI, logs, monitoring             │
│                                                                 │
│  ✅ JTI as Redis key:                                            │
│     Key: "refresh:a3f7b2c1-9e4d-4a8f-b5c6-1234abcd5678"        │
│     Benefits:                                                   │
│     ├─ Fixed length (UUID = 36 chars, always)                   │
│     ├─ Fast lookups                                             │
│     └─ Token never stored in Redis (only its ID)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key naming convention:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    KEY DESIGN                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pattern:  {namespace}:{identifier}                             │
│                                                                 │
│  Token key (one per token):                                     │
│     refresh:{jti}                                               │
│     Example: refresh:a3f7b2c1-9e4d-4a8f-b5c6-1234abcd5678      │
│     Value:   JSON with user_id, device_info, created_at         │
│     TTL:     Same as refresh token lifetime (e.g. 7 days)       │
│                                                                 │
│  User index (one per user — tracks ALL their tokens):           │
│     user_sessions:{user_id}                                     │
│     Example: user_sessions:42                                   │
│     Type:    SET of JTI strings                                 │
│     TTL:     None (maintained manually via cleanup)             │
│                                                                 │
│  Access token blocklist (only for revoked access tokens):       │
│     blocklist:{jti}                                             │
│     Example: blocklist:e8b2f1a0-...                             │
│     Value:   "1" (just a flag — existence is what matters)      │
│     TTL:     Remaining lifetime of the access token             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Think of it like the building: each badge has a serial number (JTI). The security desk keeps a list of active serial numbers (refresh:{jti}), and a list grouped by employee (user_sessions:{user_id}). If someone is fired, you look up 'all badges for employee X' and deactivate each one."

---

## 2.3 Storing and Retrieving Tokens

**Let's build the `TokenService` — the code that manages our security desk.**

> "This follows the service pattern you used in Week 6. Just like your repository separates DB logic from routes, the `TokenService` separates Redis token logic from your auth endpoints."

```python
# services/token_service.py
import json
import uuid
from datetime import datetime, timedelta
from typing import Optional

from redis.asyncio import Redis


class TokenService:
    """Manages refresh token lifecycle in Redis.
    
    This is the 'security desk' — it knows which tokens
    are alive, and can kill any of them instantly.
    """
    
    # Key prefixes — namespace to avoid collisions with cache keys
    REFRESH_PREFIX = "refresh"
    USER_SESSIONS_PREFIX = "user_sessions"
    
    def __init__(self, redis: Redis) -> None:
        self.redis = redis
    
    def _refresh_key(self, jti: str) -> str:
        """Build the Redis key for a specific refresh token."""
        return f"{self.REFRESH_PREFIX}:{jti}"
    
    def _user_sessions_key(self, user_id: int) -> str:
        """Build the Redis key for a user's token set."""
        return f"{self.USER_SESSIONS_PREFIX}:{user_id}"
    
    async def store_refresh_token(
        self,
        user_id: int,
        jti: str,
        expires_in: timedelta,
        device_info: Optional[str] = None,
    ) -> None:
        """Store a new refresh token. Called on login or token refresh.
        
        Two writes happen:
        1. The token record itself (with TTL for auto-cleanup)
        2. Add the JTI to the user's session set (for bulk revocation)
        """
        token_data = json.dumps({
            "user_id": user_id,
            "device_info": device_info,
            "created_at": datetime.utcnow().isoformat(),
        })
        
        ttl_seconds = int(expires_in.total_seconds())
        
        # Use a pipeline — both writes succeed or we notice the failure
        # (You learned pipelines in Lecture 2 for cache operations)
        async with self.redis.pipeline() as pipe:
            # 1. Store the token record with TTL
            pipe.setex(
                name=self._refresh_key(jti),
                time=ttl_seconds,
                value=token_data,
            )
            # 2. Add this JTI to the user's session set
            pipe.sadd(
                self._user_sessions_key(user_id),
                jti,
            )
            await pipe.execute()
    
    async def validate_refresh_token(self, jti: str) -> Optional[dict]:
        """Check if a refresh token is still valid.
        
        Returns the token data if valid, None if expired or revoked.
        This is the 'badge scanner' checking the security desk.
        """
        data = await self.redis.get(self._refresh_key(jti))
        
        if data is None:
            # Token doesn't exist: either expired (TTL) or revoked (DEL)
            return None
        
        return json.loads(data)
```

**What just happened — visualized:**

```
┌─────────────────────────────────────────────────────────────────┐
│              STORE REFRESH TOKEN — TWO WRITES                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User 42 logs in → server creates refresh JWT with jti="abc"    │
│                                                                 │
│  REDIS after store_refresh_token():                             │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  KEY: refresh:abc                                   │        │
│  │  VALUE: {"user_id": 42, "device_info": "Chrome",   │        │
│  │          "created_at": "2025-03-15T10:30:00"}       │        │
│  │  TTL: 604800 seconds (7 days) ← auto-expires!      │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  KEY: user_sessions:42                              │        │
│  │  TYPE: SET                                          │        │
│  │  MEMBERS: { "abc" }                                 │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│                                                                 │
│  User 42 logs in from phone → jti="def"                         │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  KEY: refresh:def                                   │        │
│  │  VALUE: {"user_id": 42, "device_info": "iOS App",  │        │
│  │          "created_at": "2025-03-15T14:00:00"}       │        │
│  │  TTL: 604800                                        │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  KEY: user_sessions:42                              │        │
│  │  TYPE: SET                                          │        │
│  │  MEMBERS: { "abc", "def" }  ← both devices tracked │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Validation flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 VALIDATE REFRESH TOKEN                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client sends refresh token JWT to /auth/refresh                │
│       │                                                         │
│       ▼                                                         │
│  Server decodes JWT → extracts jti="abc"                        │
│       │                (JWT signature check — same as Week 9)   │
│       ▼                                                         │
│  Server calls: validate_refresh_token(jti="abc")                │
│       │                                                         │
│       ▼                                                         │
│  Redis: GET refresh:abc                                         │
│       │                                                         │
│       ├── Key exists → Return token data → Issue new tokens     │
│       │                                                         │
│       └── Key missing → Token was revoked (DEL) or              │
│                         expired (TTL) → Reject with 401         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice: Redis GET returns `None` for both revoked AND expired tokens. That's elegant — we don't need separate logic. A dead token is a dead token."

---

## 2.4 Token Rotation

**Every time a refresh token is used, destroy it and issue a new one.**

> "Back to the building: when you renew your badge, the security desk SHREDS the old badge and gives you a new one with a new serial number. If someone later tries to use the old badge — that's suspicious."

**Why rotation matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHY TOKEN ROTATION?                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT rotation (reuse same refresh token):                   │
│                                                                 │
│  Attacker steals refresh token on Day 1                         │
│  Day 1: Attacker uses token → gets access ✅                    │
│  Day 2: Attacker uses SAME token → gets access ✅               │
│  Day 3: Attacker uses SAME token → gets access ✅               │
│  ...                                                            │
│  Day 7: Token expires. Attacker had 7 days of access.           │
│                                                                 │
│                                                                 │
│  WITH rotation (new refresh token on each use):                 │
│                                                                 │
│  Attacker steals refresh token on Day 1                         │
│  Day 1: Real user refreshes → old token destroyed, new issued   │
│  Day 1: Attacker tries old token → REJECTED (no longer exists)  │
│                                                                 │
│  OR:                                                            │
│  Day 1: Attacker refreshes first → old token destroyed          │
│  Day 1: Real user tries old token → REJECTED                    │
│  Day 1: Real user knows something is wrong → changes password   │
│                                                                 │
│  Either way, the window of attack shrinks dramatically.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Implementing rotation in the TokenService:**

```python
# Add this method to TokenService

    async def rotate_refresh_token(
        self,
        old_jti: str,
        new_jti: str,
        user_id: int,
        expires_in: timedelta,
        device_info: Optional[str] = None,
    ) -> bool:
        """Atomically replace an old refresh token with a new one.
        
        This is 'shred old badge, issue new badge' in one step.
        Returns True if rotation succeeded, False if old token
        was already gone (possible replay attack).
        """
        old_key = self._refresh_key(old_jti)
        new_key = self._refresh_key(new_jti)
        sessions_key = self._user_sessions_key(user_id)
        ttl_seconds = int(expires_in.total_seconds())
        
        new_token_data = json.dumps({
            "user_id": user_id,
            "device_info": device_info,
            "created_at": datetime.utcnow().isoformat(),
        })
        
        # Step 1: Check if the old token still exists
        # If it doesn't, someone already used (or revoked) it — suspicious
        old_data = await self.redis.get(old_key)
        if old_data is None:
            return False  # Old token gone — possible replay attack
        
        # Step 2: Atomic swap using a pipeline
        async with self.redis.pipeline() as pipe:
            pipe.delete(old_key)                         # Shred old badge
            pipe.setex(new_key, ttl_seconds, new_token_data)  # Issue new badge
            pipe.srem(sessions_key, old_jti)             # Remove old from set
            pipe.sadd(sessions_key, new_jti)             # Add new to set
            await pipe.execute()
        
        return True
```

**Using rotation in your refresh endpoint:**

```python
# routes/auth.py  (building on your Week 9 endpoint)

@router.post("/auth/refresh")
async def refresh_tokens(
    refresh_token: str,
    token_service: TokenService = Depends(get_token_service),
    db: AsyncSession = Depends(get_db),
) -> TokenResponse:
    # 1. Decode the refresh JWT (same as Week 9)
    payload = jwt.decode(refresh_token, SECRET_KEY, algorithms=[ALGORITHM])
    old_jti: str = payload["jti"]
    user_id: int = int(payload["sub"])
    
    # 2. Rotate: destroy old, create new
    new_jti = str(uuid.uuid4())
    rotated = await token_service.rotate_refresh_token(
        old_jti=old_jti,
        new_jti=new_jti,
        user_id=user_id,
        expires_in=timedelta(days=7),
    )
    
    if not rotated:
        # Old token already used or revoked — potential theft!
        # Safety measure: revoke ALL tokens for this user
        await token_service.revoke_all_user_tokens(user_id)
        raise HTTPException(
            status_code=401,
            detail="Token reuse detected. All sessions terminated.",
        )
    
    # 3. Issue new token pair
    new_access_token = create_access_token(user_id)
    new_refresh_token = create_refresh_token(user_id, jti=new_jti)
    
    return TokenResponse(
        access_token=new_access_token,
        refresh_token=new_refresh_token,
    )
```

**The replay detection is critical:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   REPLAY DETECTION FLOW                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Normal case:                                                   │
│     User sends refresh token (jti="abc")                        │
│     Redis: GET refresh:abc → exists ✅                           │
│     Rotate: DEL refresh:abc, SET refresh:def                    │
│     User gets new tokens with jti="def"                         │
│                                                                 │
│  Replay attack:                                                 │
│     Attacker sends SAME refresh token (jti="abc")               │
│     Redis: GET refresh:abc → None ❌ (already rotated!)          │
│     rotate returns False                                        │
│     ALARM: revoke ALL tokens for this user                      │
│     Both attacker AND user are forced to re-login               │
│     (annoying for user, but safe)                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "The user might be annoyed to re-login. But the alternative is an attacker with persistent access. Security wins."

---

## 2.5 Multi-Device Token Tracking

**The user_sessions Set gives us a powerful index.**

We already added JTIs to `user_sessions:{user_id}` in `store_refresh_token`. Now let's use that index.

```python
# Add to TokenService

    async def get_active_sessions(self, user_id: int) -> list[dict]:
        """List all active sessions for a user.
        
        Like asking the security desk: 'Show me all active
        badges for Employee #42.'
        """
        sessions_key = self._user_sessions_key(user_id)
        jti_set: set[str] = await self.redis.smembers(sessions_key)
        
        sessions = []
        stale_jtis: list[str] = []
        
        for jti_bytes in jti_set:
            jti = jti_bytes.decode() if isinstance(jti_bytes, bytes) else jti_bytes
            data = await self.redis.get(self._refresh_key(jti))
            
            if data is None:
                # Token expired (TTL) but JTI still in set — stale entry
                stale_jtis.append(jti)
                continue
            
            session = json.loads(data)
            session["jti"] = jti
            sessions.append(session)
        
        # Cleanup stale entries (tokens expired by TTL but still in the set)
        if stale_jtis:
            await self.redis.srem(sessions_key, *stale_jtis)
        
        return sessions
```

**The stale entry problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE STALE ENTRY PROBLEM                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  When a refresh token expires via TTL, Redis auto-deletes       │
│  the key refresh:{jti}. But the JTI is still sitting in         │
│  the SET at user_sessions:{user_id}.                            │
│                                                                 │
│  Redis keys:                                                    │
│  refresh:abc         ← GONE (TTL expired after 7 days)          │
│  refresh:def         ← still alive                              │
│  user_sessions:42    ← { "abc", "def" }  ← "abc" is stale!     │
│                                                                 │
│  This is harmless but wastes memory over time. We clean up      │
│  stale entries whenever we read the set (lazy cleanup). For     │
│  heavy usage, a background job (Week 11) can also prune these.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This lazy-cleanup pattern is common in Redis. Rather than running a scheduled task to sweep stale data, you clean up as you read. You'll see this again with session data later in this lecture."

---

# PART 3: TOKEN REVOCATION

## 3.1 When Revocation Matters

**Revocation solves the 2 AM incident. But it's not just about compromises.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 WHEN YOU NEED REVOCATION                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCENARIO                      │  REVOCATION ACTION             │
│  ──────────────────────────────│───────────────────────────     │
│  User clicks "Log out"         │  Revoke this session only      │
│  User clicks "Log out all      │  Revoke ALL sessions           │
│    devices"                    │                                │
│  User changes password         │  Revoke ALL sessions           │
│  Admin disables account        │  Revoke ALL + block future     │
│  Suspicious activity detected  │  Revoke ALL sessions           │
│  User role/permissions changed │  Revoke access tokens          │
│                                │  (force re-issue with new      │
│                                │   claims on next refresh)      │
│  Security breach (DB leaked)   │  Revoke EVERYTHING for         │
│                                │  ALL users (nuclear option)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Each scenario demands a different scope of revocation. Single session, all sessions for one user, or all sessions globally. Your `TokenService` must support all three."

---

## 3.2 Allowlist vs Blocklist

**Two fundamentally different philosophies for revocation.**

```
┌─────────────────────────────────────────────────────────────────┐
│               ALLOWLIST vs BLOCKLIST                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  ALLOWLIST ("only if on the list")                              │
│  ─────────────────────────────────                              │
│                                                                 │
│  Redis stores: EVERY valid token                                │
│  Validation:   Token in Redis? → Allow. Not in Redis? → Deny.  │
│  Revocation:   Delete the token from Redis.                     │
│                                                                 │
│    Login  → SADD to Redis (register the badge)                  │
│    Verify → GET from Redis (is the badge registered?)           │
│    Revoke → DEL from Redis (shred the badge)                    │
│                                                                 │
│  ✅ Simple mental model: "if it's not in Redis, it's dead"      │
│  ✅ Instant revocation: just delete the key                     │
│  ❌ EVERY request hits Redis (even normal, non-revoked tokens)  │
│  ❌ More Redis memory (stores ALL active tokens)                │
│                                                                 │
│                                                                 │
│  BLOCKLIST ("unless on the naughty list")                       │
│  ─────────────────────────────────────────                      │
│                                                                 │
│  Redis stores: ONLY revoked tokens                              │
│  Validation:   Token in blocklist? → Deny. Not there? → Allow. │
│  Revocation:   Add the token to the blocklist.                  │
│                                                                 │
│    Login  → Nothing in Redis (just issue the JWT)               │
│    Verify → Check: is it in the blocklist?                      │
│    Revoke → SET in Redis with TTL (flag as revoked)             │
│                                                                 │
│  ✅ Normal flow never touches Redis (JWT self-validates)        │
│  ✅ Less Redis memory (only revoked tokens, which are few)      │
│  ❌ Must remember to add to blocklist on every revocation event │
│  ❌ Slightly more complex: "deny if present" vs "allow if       │
│     present"                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Which one do we use?**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE HYBRID APPROACH                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REFRESH TOKENS → ALLOWLIST                                     │
│  ──────────────────────────                                     │
│  Refresh tokens are LONG-lived (days/weeks). We MUST track      │
│  them server-side. Every refresh operation already hits the     │
│  server, so one Redis lookup is negligible.                     │
│                                                                 │
│  ACCESS TOKENS → BLOCKLIST (only when needed)                   │
│  ─────────────────────────────────────────────                  │
│  Access tokens are SHORT-lived (15 min). Most of the time,      │
│  JWT signature validation is enough — no Redis needed.          │
│  We ONLY check the blocklist when we've explicitly revoked      │
│  a token (logout, security event). The blocklist is small       │
│  and entries auto-expire via TTL.                               │
│                                                                 │
│  This gives us:                                                 │
│  ├─ Full control over refresh tokens (allowlist = secure)       │
│  ├─ Minimal overhead for access tokens (blocklist = fast)       │
│  └─ Best of both worlds                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Building analogy: Long-term employee badges (refresh tokens) are registered in the security desk system — you always check the registry when they come in for the day. Short-term visitor passes (access tokens) work without checking the registry — unless a specific pass has been reported stolen, in which case it's on the 'watch list.'"

---

## 3.3 Single-Session Logout

**User clicks "Log out" — kill ONE session, leave others alive.**

```python
# Add to TokenService

    async def revoke_refresh_token(
        self,
        jti: str,
        user_id: int,
    ) -> None:
        """Revoke a single refresh token. Called on logout.
        
        Like the security desk shredding one specific badge.
        """
        async with self.redis.pipeline() as pipe:
            pipe.delete(self._refresh_key(jti))
            pipe.srem(self._user_sessions_key(user_id), jti)
            await pipe.execute()
```

**In the logout endpoint:**

```python
@router.post("/auth/logout")
async def logout(
    token: str = Depends(oauth2_scheme),
    token_service: TokenService = Depends(get_token_service),
) -> dict:
    # Decode the access token to get user info
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    user_id = int(payload["sub"])
    access_jti = payload["jti"]
    
    # 1. Blocklist the current access token (immediate effect)
    remaining_seconds = payload["exp"] - int(datetime.utcnow().timestamp())
    if remaining_seconds > 0:
        await token_service.blocklist_access_token(access_jti, remaining_seconds)
    
    # 2. Revoke the associated refresh token
    #    (The refresh JTI could be in the access token claims,
    #     or sent as a request parameter — design choice)
    if refresh_jti:
        await token_service.revoke_refresh_token(refresh_jti, user_id)
    
    return {"detail": "Logged out successfully"}
```

**What happens after logout:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   AFTER SINGLE LOGOUT                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE logout (user has 2 sessions):                           │
│                                                                 │
│  refresh:abc         → {"user_id": 42, "device": "Chrome"}     │
│  refresh:def         → {"user_id": 42, "device": "iOS"}        │
│  user_sessions:42    → { "abc", "def" }                         │
│  blocklist:          → (empty)                                  │
│                                                                 │
│                                                                 │
│  AFTER logout from Chrome (jti="abc"):                          │
│                                                                 │
│  refresh:abc         → GONE (deleted)                           │
│  refresh:def         → {"user_id": 42, "device": "iOS"}        │
│  user_sessions:42    → { "def" }  ← "abc" removed              │
│  blocklist:x7g2...   → "1" (TTL: 840s) ← access token blocked  │
│                                                                 │
│  Chrome session: dead immediately ✅                             │
│  iOS session: still alive ✅                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 All-Device Logout

**User clicks "Log out everywhere" or admin disables account.**

```python
# Add to TokenService

    async def revoke_all_user_tokens(self, user_id: int) -> int:
        """Revoke ALL refresh tokens for a user. Called on:
        - 'Log out all devices'
        - Password change
        - Account compromise
        - Admin account disable
        
        Like telling security: 'Deactivate EVERY badge
        issued to Employee #42. All of them. Now.'
        
        Returns the number of tokens revoked.
        """
        sessions_key = self._user_sessions_key(user_id)
        
        # Get all JTIs for this user
        jti_set: set[bytes] = await self.redis.smembers(sessions_key)
        
        if not jti_set:
            return 0
        
        # Build list of all keys to delete
        refresh_keys = [
            self._refresh_key(
                jti.decode() if isinstance(jti, bytes) else jti
            )
            for jti in jti_set
        ]
        
        # Atomic: delete all token keys + the sessions set itself
        async with self.redis.pipeline() as pipe:
            pipe.delete(*refresh_keys)        # Shred all badges
            pipe.delete(sessions_key)          # Clear the employee's badge list
            await pipe.execute()
        
        return len(jti_set)
```

**Visualized:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 ALL-DEVICE LOGOUT                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE (user has 3 active sessions):                           │
│                                                                 │
│  refresh:abc  → {"device": "Chrome"}    ──┐                     │
│  refresh:def  → {"device": "iOS"}        ─┤ All belong to       │
│  refresh:ghi  → {"device": "Android"}   ──┤ user 42             │
│  user_sessions:42 → { "abc", "def", "ghi" }                    │
│                                                                 │
│                                                                 │
│  revoke_all_user_tokens(user_id=42)                             │
│       │                                                         │
│       ▼                                                         │
│  Pipeline:                                                      │
│    DEL refresh:abc refresh:def refresh:ghi                      │
│    DEL user_sessions:42                                         │
│                                                                 │
│                                                                 │
│  AFTER:                                                         │
│                                                                 │
│  refresh:abc         → GONE                                     │
│  refresh:def         → GONE                                     │
│  refresh:ghi         → GONE                                     │
│  user_sessions:42    → GONE                                     │
│                                                                 │
│  Every device: next refresh attempt → 401 → forced re-login ✅  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Access Token Blocklisting

**For immediate revocation of the short-lived access token too.**

> "Revoking refresh tokens prevents the attacker from getting NEW access tokens. But what about the access token they ALREADY HAVE? It's valid for up to 15 more minutes. Sometimes that window is too long."

```python
# Add to TokenService

    BLOCKLIST_PREFIX = "blocklist"
    
    def _blocklist_key(self, jti: str) -> str:
        return f"{self.BLOCKLIST_PREFIX}:{jti}"
    
    async def blocklist_access_token(
        self,
        jti: str,
        expires_in: int,
    ) -> None:
        """Add an access token to the blocklist.
        
        The TTL is set to the token's REMAINING lifetime.
        Once the token would have expired naturally, the
        blocklist entry auto-deletes — no wasted memory.
        
        Like adding a badge to the 'stolen badges' clipboard
        at the security desk. The note auto-shreds when the
        badge would have expired anyway.
        """
        if expires_in > 0:
            await self.redis.setex(
                name=self._blocklist_key(jti),
                time=expires_in,
                value="1",  # Value doesn't matter — existence is the signal
            )
    
    async def is_access_token_blocklisted(self, jti: str) -> bool:
        """Check if an access token has been revoked.
        
        Returns True if blocked, False if allowed.
        """
        return await self.redis.exists(self._blocklist_key(jti)) > 0
```

**Why TTL on the blocklist entry matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│            BLOCKLIST TTL = REMAINING TOKEN LIFE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Access token issued at 10:00 AM, expires at 10:15 AM          │
│  User logs out at 10:08 AM                                      │
│                                                                 │
│  Remaining lifetime: 7 minutes = 420 seconds                    │
│                                                                 │
│  blocklist:x7g2... → "1"  (TTL: 420 seconds)                   │
│                                                                 │
│  10:08–10:15 AM: blocklist entry exists → token rejected        │
│  10:15 AM:       blocklist entry auto-expires                   │
│                  token would have expired naturally too          │
│                                                                 │
│  Net effect: no memory waste. The blocklist entry lives         │
│  EXACTLY as long as it needs to. After that, the token is       │
│  dead by its own expiry anyway — no need to remember it.        │
│                                                                 │
│  If you DON'T set a TTL, the blocklist grows forever.           │
│  Millions of entries for tokens that expired months ago.        │
│  TTL makes the blocklist self-cleaning.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.6 Revocation in Your FastAPI Dependencies

**Now let's wire this into the dependency chain you built in Week 9.**

> "In Week 9 you created a `get_current_user` dependency that decodes the JWT and fetches the user from PostgreSQL. Now we add ONE line: check the blocklist before accepting the token."

**Before (Week 9 — JWT only):**

```python
# dependencies/auth.py — your Week 9 version

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    user_id = int(payload["sub"])
    
    user = await user_repository.get_by_id(db, user_id)
    if user is None:
        raise HTTPException(status_code=401, detail="User not found")
    
    return user
```

**After (Week 10 — JWT + Redis blocklist):**

```python
# dependencies/auth.py — updated with blocklist check

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
    token_service: TokenService = Depends(get_token_service),
) -> User:
    # Step 1: Decode JWT (unchanged from Week 9)
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
    
    # Step 2: NEW — Check if this access token was revoked
    jti = payload.get("jti")
    if jti and await token_service.is_access_token_blocklisted(jti):
        raise HTTPException(status_code=401, detail="Token has been revoked")
    
    # Step 3: Fetch user from DB (unchanged from Week 9)
    user_id = int(payload["sub"])
    user = await user_repository.get_by_id(db, user_id)
    if user is None:
        raise HTTPException(status_code=401, detail="User not found")
    
    return user
```

**The flow with blocklist check:**

```
┌─────────────────────────────────────────────────────────────────┐
│           REQUEST AUTHENTICATION FLOW (UPDATED)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client Request                                                 │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────┐                                           │
│  │ Decode JWT       │ ← Same as Week 9: verify signature,      │
│  │ (signature check)│   check expiry. Stateless, fast.          │
│  └────────┬─────────┘                                           │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────┐                                           │
│  │ Check blocklist  │ ← NEW: Redis EXISTS check. O(1).         │
│  │ (Redis EXISTS)   │   ~0.1ms. Only catches revoked tokens.   │
│  └────────┬─────────┘                                           │
│           │                                                     │
│       ┌───┴───┐                                                 │
│       │       │                                                 │
│    Blocked  Not blocked                                         │
│       │       │                                                 │
│       ▼       ▼                                                 │
│    401 ❌   ┌──────────────────┐                                 │
│            │ Fetch user from  │ ← Same as Week 9: get user      │
│            │ PostgreSQL       │   for route handler to use.      │
│            └────────┬─────────┘                                  │
│                     │                                            │
│                     ▼                                            │
│               Route handler runs ✅                              │
│                                                                 │
│  Normal requests: JWT decode + Redis EXISTS = ~0.2ms overhead   │
│  Revoked tokens:  Caught at step 2, never hit PostgreSQL        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight:**

> "For normal (non-revoked) tokens, the Redis EXISTS check returns 0 in sub-millisecond time. It's essentially free. But when you NEED to revoke a token, it catches it instantly. You pay almost nothing for the ability to pull the emergency brake."

---

# PART 4: DISTRIBUTED RATE LIMITING

## 4.1 The Multi-Instance Problem

> "In Week 9, you learned that auth endpoints need rate limiting to prevent brute-force attacks. But we never answered: WHERE does the counter live?"

```
┌─────────────────────────────────────────────────────────────────┐
│              THE MULTI-INSTANCE PROBLEM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SINGLE SERVER (development):                                   │
│                                                                 │
│  Client ──▶ [Server]                                            │
│              counter: {"user42": 3 attempts}                    │
│              ↑ in-memory dict works fine here                   │
│                                                                 │
│                                                                 │
│  MULTIPLE SERVERS (production):                                 │
│                                                                 │
│  Client ──▶ [Load Balancer]                                     │
│                 │                                               │
│          ┌──────┼──────┐                                        │
│          ▼      ▼      ▼                                        │
│      [Server1] [Server2] [Server3]                              │
│      counter:  counter:  counter:                               │
│      user42:1  user42:1  user42:1                               │
│                                                                 │
│      Each server sees 1 attempt.                                │
│      Actual attempts: 3.                                        │
│      Rate limit (5/min)? Never triggered!                       │
│                                                                 │
│      Attacker sends 4 requests to each server = 12 total,      │
│      but no single server sees more than 4. Limit bypassed.    │
│                                                                 │
│                                                                 │
│  SOLUTION: Shared counter in Redis                              │
│                                                                 │
│  Client ──▶ [Load Balancer]                                     │
│                 │                                               │
│          ┌──────┼──────┐                                        │
│          ▼      ▼      ▼                                        │
│      [Server1] [Server2] [Server3]                              │
│          │         │         │                                  │
│          └─────────┼─────────┘                                  │
│                    ▼                                             │
│               [  Redis  ]                                       │
│               user42: 3  ← single source of truth               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Building analogy: imagine three entrances to the building, each with its own guard. If guards don't share a radio, an intruder can try 5 codes at the north door, 5 at the south door, and 5 at the east door — 15 attempts, no alarm. Give them a shared radio (Redis), and every attempt is counted centrally."

---

## 4.2 Fixed Window Counter (INCR + EXPIRE)

**The simplest pattern. Handles 90% of rate limiting needs.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  FIXED WINDOW COUNTER                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Concept: Count requests in a time window.                      │
│           If count > limit, reject.                             │
│                                                                 │
│  Window: 1 minute                                               │
│  Limit:  5 requests                                             │
│                                                                 │
│  Time: 10:00        10:01        10:02                          │
│        │            │            │                              │
│        ├─ req1 ✅    ├─ req1 ✅    │                              │
│        ├─ req2 ✅    ├─ req2 ✅    │  (counter resets each        │
│        ├─ req3 ✅    ├─ req3 ✅    │   minute window)             │
│        ├─ req4 ✅    │            │                              │
│        ├─ req5 ✅    │            │                              │
│        ├─ req6 ❌    │            │                              │
│        ├─ req7 ❌    │            │                              │
│                                                                 │
│  Redis key:  ratelimit:login:user42:10:00                       │
│              ↑ auto-expires after 60 seconds                    │
│                                                                 │
│  Next window gets a NEW key (different minute):                 │
│              ratelimit:login:user42:10:01                        │
│              ↑ counter starts at 0 again                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The implementation — two Redis commands:**

```python
# services/rate_limiter.py
from redis.asyncio import Redis


class RateLimiter:
    """Distributed rate limiter using Redis counters.
    
    Like the shared radio between building security guards:
    every entrance reports every attempt to one central log.
    """
    
    PREFIX = "ratelimit"
    
    def __init__(self, redis: Redis) -> None:
        self.redis = redis
    
    async def check_fixed_window(
        self,
        identifier: str,
        max_requests: int,
        window_seconds: int,
    ) -> tuple[bool, int]:
        """Check if request is within rate limit.
        
        Args:
            identifier: Who is being limited (e.g., "login:user42" 
                        or "login:192.168.1.1")
            max_requests: Maximum requests allowed in the window
            window_seconds: Window size in seconds
        
        Returns:
            (allowed, current_count)
            allowed: True if under limit, False if over
            current_count: How many requests so far in this window
        """
        # Compute the current window boundary
        # e.g., for 60s windows: timestamp 1710500075 → window 1710500040
        now = int(time.time())
        window_start = (now // window_seconds) * window_seconds
        
        key = f"{self.PREFIX}:{identifier}:{window_start}"
        
        # INCR: atomically increment counter. Creates key with value 1
        #        if it doesn't exist. (You learned this in Lecture 1)
        current_count = await self.redis.incr(key)
        
        if current_count == 1:
            # First request in this window — set the TTL
            # The key will auto-delete after the window passes
            await self.redis.expire(key, window_seconds)
        
        allowed = current_count <= max_requests
        return allowed, current_count
```

**Let's trace through an example:**

```python
# Login attempt from user42 at 10:00:05
limiter = RateLimiter(redis)

# Request 1:
allowed, count = await limiter.check_fixed_window("login:user42", 5, 60)
# window_start = 10:00:00
# key = "ratelimit:login:user42:1710500400"
# INCR → 1 (key created)
# EXPIRE 60 (key dies at 10:01:00)
# count=1, allowed=True ✅

# Request 2 (10:00:10):
allowed, count = await limiter.check_fixed_window("login:user42", 5, 60)
# INCR → 2 (same key, count goes up)
# count=2, allowed=True ✅

# ... Requests 3, 4, 5 ...

# Request 6 (10:00:45):
allowed, count = await limiter.check_fixed_window("login:user42", 5, 60)
# INCR → 6
# count=6, allowed=False ❌ → Return 429 Too Many Requests

# Request at 10:01:05 (new window):
allowed, count = await limiter.check_fixed_window("login:user42", 5, 60)
# window_start = 10:01:00
# key = "ratelimit:login:user42:1710500460"  ← NEW key!
# INCR → 1 (fresh start)
# count=1, allowed=True ✅
```

**The edge case — the boundary spike:**

```
┌─────────────────────────────────────────────────────────────────┐
│               FIXED WINDOW BOUNDARY PROBLEM                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Limit: 5 requests per minute                                   │
│                                                                 │
│  Window 1 (10:00-10:01)      Window 2 (10:01-10:02)            │
│       │                           │                             │
│       │              5 reqs here  │ 5 reqs here                 │
│       │              (10:00:50-   │ (10:01:00-                  │
│       │               10:00:59)   │  10:01:09)                  │
│       │                    ▼      │  ▼                          │
│       │              ─────[#####][#####]─────                   │
│       │                         ↑                               │
│       │               10 requests in 19 seconds!                │
│       │               But each window only sees 5.              │
│       │               Limit "bypassed" at the boundary.         │
│                                                                 │
│  This is a KNOWN tradeoff of fixed windows.                     │
│  For auth rate limiting, it's usually acceptable —              │
│  the attacker gets at most 2x the limit briefly.               │
│  For stricter needs, use the sliding window (next section).     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Sliding Window (Sorted Sets)

**For stricter rate limiting, track EVERY request timestamp.**

> "Recall from Lecture 1: Sorted Sets store members with a score. We use the timestamp as the score, giving us a time-ordered log of every request."

```
┌─────────────────────────────────────────────────────────────────┐
│                SLIDING WINDOW CONCEPT                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Instead of fixed buckets, the window SLIDES with time.         │
│  "How many requests in the LAST 60 seconds from NOW?"           │
│                                                                 │
│  Time: ──────────────────────────────────────────▶              │
│                                                                 │
│  At 10:00:30, window = [09:59:31 ... 10:00:30]                  │
│        │                                                        │
│        ▼                                                        │
│  ──────[############### 60 seconds ###############]──────       │
│         ↑                                          ↑            │
│        now - 60s                                  now           │
│                                                                 │
│  At 10:00:45, window = [09:59:46 ... 10:00:45]                  │
│        │                                                        │
│        ▼                                                        │
│  ────────────[############### 60 seconds ###############]───    │
│               ↑                                          ↑      │
│              now - 60s                                  now     │
│                                                                 │
│  The window slides forward with each request.                   │
│  No boundary spikes — every check is "last N seconds."          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation with Sorted Sets:**

```python
# Add to RateLimiter class

    async def check_sliding_window(
        self,
        identifier: str,
        max_requests: int,
        window_seconds: int,
    ) -> tuple[bool, int]:
        """Sliding window rate limiter using Redis Sorted Sets.
        
        Each request is a member with its timestamp as the score.
        We count members within the last `window_seconds`.
        
        More accurate than fixed window, but uses more memory
        (one entry per request vs one counter per window).
        """
        key = f"{self.PREFIX}:sliding:{identifier}"
        now = time.time()
        window_start = now - window_seconds
        
        async with self.redis.pipeline() as pipe:
            # 1. Remove entries older than the window
            #    (cleanup: anything before window_start is irrelevant)
            pipe.zremrangebyscore(key, 0, window_start)
            
            # 2. Add current request (use timestamp as both score and member)
            #    Use a unique member to avoid deduplication:
            request_id = f"{now}:{uuid.uuid4().hex[:8]}"
            pipe.zadd(key, {request_id: now})
            
            # 3. Count how many entries are in the window
            pipe.zcard(key)
            
            # 4. Set TTL on the key (auto-cleanup if user goes quiet)
            pipe.expire(key, window_seconds)
            
            results = await pipe.execute()
        
        # results[2] is the ZCARD result (count of entries)
        current_count = results[2]
        allowed = current_count <= max_requests
        
        return allowed, current_count
```

**Tracing through the Sorted Set state:**

```
┌─────────────────────────────────────────────────────────────────┐
│           SORTED SET STATE OVER TIME                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Limit: 3 requests per 60 seconds                               │
│                                                                 │
│  10:00:10 — Request 1                                           │
│  Key: ratelimit:sliding:login:user42                            │
│  Sorted Set: { (10:00:10, "10.00.10:a3f7") }                   │
│  Count: 1 → allowed ✅                                          │
│                                                                 │
│  10:00:25 — Request 2                                           │
│  Sorted Set: { (10:00:10, "..."), (10:00:25, "...") }          │
│  Count: 2 → allowed ✅                                          │
│                                                                 │
│  10:00:40 — Request 3                                           │
│  Sorted Set: { (10:00:10, "..."), (10:00:25, "..."),            │
│                (10:00:40, "...") }                               │
│  Count: 3 → allowed ✅                                          │
│                                                                 │
│  10:00:55 — Request 4                                           │
│  Sorted Set: { ..., (10:00:55, "...") }                         │
│  Count: 4 → NOT allowed ❌ (4 > 3)                              │
│                                                                 │
│  10:01:15 — Request 5                                           │
│  Step 1: ZREMRANGEBYSCORE removes entries before 10:00:15       │
│          → entry at 10:00:10 is removed                         │
│  Sorted Set: { (10:00:25, "..."), (10:00:40, "..."),            │
│                (10:00:55, "..."), (10:01:15, "...") }           │
│  Count: 4 → NOT allowed ❌                                      │
│                                                                 │
│  10:01:30 — Request 6                                           │
│  ZREMRANGEBYSCORE removes entries before 10:00:30               │
│  → entry at 10:00:25 removed                                    │
│  Sorted Set: { (10:00:40, "..."), (10:00:55, "..."),            │
│                (10:01:15, "..."), (10:01:30, "...") }           │
│  Count: 4 → NOT allowed ❌                                      │
│  (Request from 10:00:40 hasn't "expired" yet)                   │
│                                                                 │
│  10:01:45 — Request 7                                           │
│  ZREMRANGEBYSCORE removes entries before 10:00:45               │
│  → entries at 10:00:40 removed                                  │
│  Sorted Set: { (10:00:55, "..."), (10:01:15, "..."),            │
│                (10:01:30, "..."), (10:01:45, "...") }           │
│  Count: 4 → NOT allowed ❌                                      │
│  (10:00:55 still in window — 50 seconds ago)                    │
│                                                                 │
│  No boundary spike. The window genuinely slides.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Fixed Window vs Sliding Window:**

```
┌─────────────────────────────────────────────────────────────────┐
│                WHICH ONE TO USE?                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                  Fixed Window        Sliding Window             │
│  ─────────────── ─────────────────── ──────────────────         │
│  Redis commands  1 INCR + 1 EXPIRE   ZREMRANGE + ZADD +        │
│  per request                         ZCARD + EXPIRE             │
│                                                                 │
│  Memory per user 1 small key         1 entry per request        │
│                                      in the window              │
│                                                                 │
│  Accuracy        Boundary spike      Exact count                │
│                  possible                                       │
│                                                                 │
│  Speed           Faster (2 ops)      Slower (4 ops)             │
│                                                                 │
│  Use when:       Most cases.         You need strict            │
│                  Login rate limit.    accuracy. Billing          │
│                  API rate limit.      APIs. Security-critical    │
│                                      endpoints.                 │
│                                                                 │
│  For this course: Fixed window for login rate limiting.         │
│  You now understand both — use sliding when you need it.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Rate Limiter as FastAPI Dependency

**Wire the rate limiter into your auth endpoints.**

```python
# dependencies/rate_limit.py
from fastapi import Depends, HTTPException, Request

from services.rate_limiter import RateLimiter
from dependencies.redis import get_redis


async def get_rate_limiter(
    redis: Redis = Depends(get_redis),
) -> RateLimiter:
    return RateLimiter(redis)


def rate_limit(
    max_requests: int,
    window_seconds: int,
    key_func: str = "ip",
):
    """Factory that creates a rate-limiting dependency.
    
    Usage:
        @router.post("/auth/login", dependencies=[Depends(rate_limit(5, 60))])
    
    key_func options:
        "ip"   — limit per IP address (for login, registration)
        "user" — limit per authenticated user (for API usage)
    """
    async def _check_rate_limit(
        request: Request,
        limiter: RateLimiter = Depends(get_rate_limiter),
    ) -> None:
        # Determine who we're rate limiting
        if key_func == "ip":
            identifier = f"ip:{request.client.host}"
        else:
            # For user-based, the caller must be authenticated
            # (use after auth dependency)
            identifier = f"user:{request.state.user_id}"
        
        allowed, count = await limiter.check_fixed_window(
            identifier=identifier,
            max_requests=max_requests,
            window_seconds=window_seconds,
        )
        
        if not allowed:
            raise HTTPException(
                status_code=429,
                detail="Too many requests",
                headers={
                    "Retry-After": str(window_seconds),
                    "X-RateLimit-Limit": str(max_requests),
                    "X-RateLimit-Remaining": "0",
                    "X-RateLimit-Reset": str(
                        int(time.time()) + window_seconds
                    ),
                },
            )
    
    return _check_rate_limit
```

**Using it on your auth routes:**

```python
# routes/auth.py

@router.post(
    "/auth/login",
    dependencies=[Depends(rate_limit(max_requests=5, window_seconds=60))],
)
async def login(credentials: LoginRequest, ...) -> TokenResponse:
    ...

@router.post(
    "/auth/register",
    dependencies=[Depends(rate_limit(max_requests=3, window_seconds=300))],
)
async def register(user_data: RegisterRequest, ...) -> UserResponse:
    ...
```

> "Notice the response headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`. You learned about these header conventions in Week 4, Lecture 2 — API design principles. We're implementing what you were taught to respect as a client."

---

# PART 5: SESSION DATA STORAGE

## 5.1 Sessions vs JWT Claims

> "You've been storing user info in JWT claims — `sub`, `exp`, `role`. But JWT claims are frozen at creation time. What about data that changes BETWEEN requests?"

```
┌─────────────────────────────────────────────────────────────────┐
│              JWT CLAIMS vs SESSION DATA                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JWT CLAIMS (baked in at token creation)                        │
│  ──────────────────────────────────────                         │
│  Set once, read many times. Cannot change without new token.    │
│                                                                 │
│  Good for:                                                      │
│  ├─ user_id (doesn't change within a session)                   │
│  ├─ role (changes rarely — on next refresh, get updated claims) │
│  └─ token_type, expiry, issued_at (metadata)                    │
│                                                                 │
│                                                                 │
│  SESSION DATA (live in Redis, changes anytime)                  │
│  ─────────────────────────────────────────────                  │
│  Read and written throughout the session lifetime.              │
│                                                                 │
│  Good for:                                                      │
│  ├─ last_active_at (updated every request — "online" status)    │
│  ├─ current_org_id (user switches between orgs — multi-tenant)  │
│  ├─ active_project_id (user navigates between projects)         │
│  ├─ csrf_token (rotated periodically for security)              │
│  └─ cached_permissions (avoid DB lookup every request, but      │
│     must update when permissions change)                        │
│                                                                 │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  ├─ If it's IDENTITY → JWT claim                                │
│  ├─ If it changes during the session → Redis session data       │
│  └─ If it's large or complex → don't put it in either           │
│     (fetch from DB when needed)                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Redis Hash for Session Data

> "In Lecture 1 you learned Hashes — a Redis key that holds a set of field-value pairs. That's exactly what a session is: a bag of attributes for one user's current state."

```python
# Add to TokenService (or create a separate SessionService)

    SESSION_PREFIX = "session"
    
    def _session_key(self, user_id: int) -> str:
        return f"{self.SESSION_PREFIX}:{user_id}"
    
    async def set_session_data(
        self,
        user_id: int,
        data: dict[str, str],
        ttl: int = 3600,  # 1 hour of inactivity → session expires
    ) -> None:
        """Store or update session fields.
        
        Uses HSET — only updates the fields you pass, leaves
        existing fields untouched. (Unlike SET, which replaces.)
        """
        key = self._session_key(user_id)
        
        async with self.redis.pipeline() as pipe:
            pipe.hset(key, mapping=data)
            pipe.expire(key, ttl)  # Reset TTL on every update
            await pipe.execute()
    
    async def get_session_data(
        self,
        user_id: int,
    ) -> Optional[dict[str, str]]:
        """Get all session fields for a user."""
        key = self._session_key(user_id)
        data = await self.redis.hgetall(key)
        
        if not data:
            return None
        
        # Redis returns bytes — decode to strings
        return {
            k.decode() if isinstance(k, bytes) else k:
            v.decode() if isinstance(v, bytes) else v
            for k, v in data.items()
        }
    
    async def get_session_field(
        self,
        user_id: int,
        field: str,
    ) -> Optional[str]:
        """Get a single session field. Cheaper than HGETALL
        when you only need one value."""
        key = self._session_key(user_id)
        value = await self.redis.hget(key, field)
        if value is None:
            return None
        return value.decode() if isinstance(value, bytes) else value
    
    async def delete_session(self, user_id: int) -> None:
        """Destroy a session entirely. Called on logout."""
        await self.redis.delete(self._session_key(user_id))
```

**What the session looks like in Redis:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SESSION HASH IN REDIS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  KEY: session:42                                                │
│  TYPE: HASH                                                     │
│  TTL: 3600 (resets on every update — sliding expiry)            │
│                                                                 │
│  ┌────────────────────┬──────────────────────────────┐          │
│  │  FIELD             │  VALUE                       │          │
│  ├────────────────────┼──────────────────────────────┤          │
│  │  last_active_at    │  "2025-03-15T14:22:08"       │          │
│  │  current_org_id    │  "7"                         │          │
│  │  active_project_id │  "23"                        │          │
│  │  login_ip          │  "203.0.113.42"              │          │
│  │  login_device      │  "Chrome/120 macOS"          │          │
│  └────────────────────┴──────────────────────────────┘          │
│                                                                 │
│  HSET session:42 last_active_at "2025-03-15T14:22:08"          │
│     → updates ONE field, leaves others intact                   │
│                                                                 │
│  HGET session:42 current_org_id                                 │
│     → returns "7" (O(1) lookup)                                 │
│                                                                 │
│  HGETALL session:42                                             │
│     → returns all fields (for session overview / admin panel)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Session Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                   SESSION LIFECYCLE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LOGIN                                                          │
│  ─────                                                          │
│  User authenticates → create session + tokens                   │
│                                                                 │
│    HSET session:42 last_active_at "..." login_ip "..."          │
│    SETEX refresh:abc ... (token stored)                         │
│    EXPIRE session:42 3600 (1hr inactivity timeout)              │
│                                                                 │
│                                                                 │
│  DURING USAGE                                                   │
│  ────────────                                                   │
│  Every authenticated request → update last_active, reset TTL    │
│                                                                 │
│    HSET session:42 last_active_at "..."                         │
│    EXPIRE session:42 3600 (reset the clock)                     │
│                                                                 │
│  User switches organization:                                    │
│    HSET session:42 current_org_id "12"                          │
│                                                                 │
│                                                                 │
│  INACTIVITY                                                     │
│  ──────────                                                     │
│  If user does nothing for 1 hour → TTL expires → session gone   │
│  Next request → no session → optionally rebuild from DB         │
│                                                                 │
│                                                                 │
│  LOGOUT                                                         │
│  ──────                                                         │
│  Explicit logout → destroy everything                           │
│                                                                 │
│    DEL session:42 (session data gone)                           │
│    DEL refresh:abc (token revoked)                              │
│    SETEX blocklist:xyz 900 "1" (access token blocked)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Session activity tracking as middleware:**

```python
# middleware/session_activity.py

from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from datetime import datetime


class SessionActivityMiddleware(BaseHTTPMiddleware):
    """Update session last_active on every authenticated request.
    
    This keeps the session alive while the user is active,
    and lets the TTL kill it when they go idle.
    """
    
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # Only update if the request was authenticated
        # (user_id is set by the auth dependency)
        user_id = getattr(request.state, "user_id", None)
        if user_id is not None:
            token_service: TokenService = request.app.state.token_service
            await token_service.set_session_data(
                user_id=user_id,
                data={"last_active_at": datetime.utcnow().isoformat()},
            )
        
        return response
```

> "Building analogy: session data is like the employee preference card at the security desk. 'Employee #42 is currently on Floor 7, working on Project Alpha, last scanned in at 2:15 PM.' It's not on the badge — it's at the desk, and it updates throughout the day."

---

# PART 6: GRACEFUL DEGRADATION

## 6.1 When Redis Goes Down

**Redis is fast. Redis is useful. Redis will, at some point, be unreachable.**

> "In Week 8, you learned about the circuit breaker pattern — detecting when an external service is failing and failing fast instead of waiting. Redis IS one of those external services. What happens to your auth system when it vanishes?"

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT BREAKS WHEN REDIS DIES                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FEATURE                   │  WITHOUT REDIS                     │
│  ──────────────────────────│─────────────────────────────       │
│  JWT signature validation  │  ✅ WORKS (no Redis needed)        │
│                            │  JWT is self-contained.            │
│                            │                                    │
│  Access token blocklist    │  ⚠️ BROKEN — can't check if        │
│                            │  token was revoked.                │
│                            │                                    │
│  Refresh token validation  │  ❌ BROKEN — can't verify token    │
│                            │  exists in allowlist.              │
│                            │                                    │
│  Token rotation            │  ❌ BROKEN — can't delete old      │
│                            │  or store new.                     │
│                            │                                    │
│  Rate limiting             │  ❌ BROKEN — counters lost.        │
│                            │  No limit enforcement.             │
│                            │                                    │
│  Session data              │  ❌ BROKEN — session fields        │
│                            │  unavailable.                      │
│                            │                                    │
│  Cache (Lecture 2)         │  ❌ BROKEN — cache misses,         │
│                            │  all requests hit DB.              │
│                            │                                    │
└─────────────────────────────────────────────────────────────────┘
```

> "The building analogy: a power outage at the security desk. Badge scanners at the doors still work (JWT signature check). But the security desk can't check its registry (Redis allowlist), can't check the 'stolen badges' clipboard (blocklist), and can't count scan attempts (rate limiting). What's your policy? Lock everyone out? Or let badges work but accept the risk that a revoked badge might get through?"

**This is a business decision, not a purely technical one.**

---

## 6.2 Fallback Strategies by Feature

```
┌─────────────────────────────────────────────────────────────────┐
│            DEGRADATION STRATEGY PER FEATURE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  ACCESS TOKEN BLOCKLIST CHECK                                   │
│  ─────────────────────────────                                  │
│  Fallback: SKIP the blocklist check, accept JWT as valid.       │
│  Risk:     Revoked tokens work for up to 15 minutes.            │
│  Why OK:   Access tokens are short-lived. The window is small.  │
│            Users can still function. Service stays up.           │
│                                                                 │
│                                                                 │
│  REFRESH TOKEN VALIDATION                                       │
│  ─────────────────────────                                      │
│  Fallback: REJECT all refresh attempts.                         │
│  Risk:     Users must re-login when their access token expires. │
│  Why OK:   Allowing unverified refreshes is too dangerous.      │
│            Re-login is annoying but safe.                        │
│                                                                 │
│                                                                 │
│  RATE LIMITING                                                  │
│  ─────────────                                                  │
│  Fallback: Switch to in-memory counter (per-instance, not       │
│            distributed). Or disable temporarily.                │
│  Risk:     Limits are per-instance, not global. Brute-force     │
│            protection is weakened but not gone.                  │
│  Why OK:   Some limiting > no limiting.                         │
│                                                                 │
│                                                                 │
│  SESSION DATA                                                   │
│  ────────────                                                   │
│  Fallback: Use defaults. Or fall back to DB query.              │
│  Risk:     Missing preferences, stale org context.              │
│  Why OK:   Non-critical data. Slight UX degradation.            │
│                                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.3 Implementing Degradation

**Wrap Redis calls with try/except and fall back gracefully.**

```python
# Updated get_current_user with graceful degradation

from redis.exceptions import RedisError

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
    token_service: TokenService = Depends(get_token_service),
) -> User:
    # Step 1: Decode JWT — no Redis needed, always works
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
    
    # Step 2: Check blocklist — with fallback
    jti = payload.get("jti")
    if jti:
        try:
            if await token_service.is_access_token_blocklisted(jti):
                raise HTTPException(
                    status_code=401,
                    detail="Token has been revoked",
                )
        except RedisError:
            # Redis is down — we CANNOT check the blocklist.
            # Decision: SKIP the check, let the request through.
            # Log this so we know degradation is happening.
            logger.warning(
                "Redis unavailable — blocklist check skipped",
                extra={"jti": jti, "user_id": payload.get("sub")},
            )
            # The JWT signature is still valid. We accept the risk
            # that a recently-revoked token might get through.
    
    # Step 3: Fetch user from DB — no Redis needed
    user_id = int(payload["sub"])
    user = await user_repository.get_by_id(db, user_id)
    if user is None:
        raise HTTPException(status_code=401, detail="User not found")
    
    return user
```

**Rate limiter with fallback:**

```python
# Updated rate limiter with graceful degradation
import logging

logger = logging.getLogger(__name__)

# In-memory fallback counter (per-instance, not distributed)
_fallback_counters: dict[str, dict] = {}


async def _check_rate_limit(
    request: Request,
    limiter: RateLimiter = Depends(get_rate_limiter),
) -> None:
    identifier = f"ip:{request.client.host}"
    
    try:
        allowed, count = await limiter.check_fixed_window(
            identifier=identifier,
            max_requests=max_requests,
            window_seconds=window_seconds,
        )
    except RedisError:
        # Redis down — fall back to in-memory (per-instance) limiting
        logger.warning("Redis unavailable — using in-memory rate limit")
        allowed, count = _check_in_memory_fallback(
            identifier, max_requests, window_seconds
        )
    
    if not allowed:
        raise HTTPException(status_code=429, detail="Too many requests")


def _check_in_memory_fallback(
    identifier: str,
    max_requests: int,
    window_seconds: int,
) -> tuple[bool, int]:
    """Simple in-memory counter as fallback.
    Not distributed — only limits per this server instance.
    Better than no limit at all.
    """
    now = time.time()
    
    if identifier not in _fallback_counters:
        _fallback_counters[identifier] = {"count": 0, "window_start": now}
    
    entry = _fallback_counters[identifier]
    
    # Reset if window has passed
    if now - entry["window_start"] > window_seconds:
        entry["count"] = 0
        entry["window_start"] = now
    
    entry["count"] += 1
    return entry["count"] <= max_requests, entry["count"]
```

---

## 6.4 The Degradation Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│           DEGRADATION DECISION FRAMEWORK                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  For each feature that depends on Redis, ask:                   │
│                                                                 │
│         ┌───────────────────────────────┐                       │
│         │ If Redis is unreachable,      │                       │
│         │ what happens if I SKIP this?  │                       │
│         └──────────────┬────────────────┘                       │
│                        │                                        │
│              ┌─────────┴─────────┐                              │
│              │                   │                              │
│          Safety risk         Convenience risk                   │
│          (security hole)     (UX degradation)                   │
│              │                   │                              │
│              ▼                   ▼                              │
│     ┌──────────────────┐  ┌──────────────────┐                  │
│     │ CAN you fall     │  │ SKIP it and       │                 │
│     │ back to a safe   │  │ log a warning.    │                 │
│     │ default?         │  │ User barely       │                 │
│     └────────┬─────────┘  │ notices.          │                 │
│              │            └──────────────────┘                  │
│         ┌────┴────┐                                             │
│         │         │                                             │
│        YES        NO                                            │
│         │         │                                             │
│         ▼         ▼                                             │
│  ┌────────────┐ ┌──────────────┐                                │
│  │ Use the    │ │ REJECT the   │                                │
│  │ fallback.  │ │ request.     │                                │
│  │ e.g. skip  │ │ Return 503.  │                                │
│  │ blocklist  │ │ e.g. reject  │                                │
│  │ check.     │ │ all refresh  │                                │
│  └────────────┘ │ attempts.    │                                │
│                 └──────────────┘                                │
│                                                                 │
│  ALWAYS LOG DEGRADATION EVENTS.                                 │
│  You need to know it's happening to fix Redis.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How the health check should reflect Redis status:**

```python
# routes/health.py

@router.get("/health")
async def health_check(
    redis: Redis = Depends(get_redis),
    db: AsyncSession = Depends(get_db),
) -> dict:
    """Health endpoint for load balancers and monitoring.
    
    Returns 200 if core services are up.
    Includes Redis status so operators know when
    degradation is active.
    """
    health = {"status": "healthy", "checks": {}}
    
    # Check PostgreSQL (critical — if DB is down, nothing works)
    try:
        await db.execute(text("SELECT 1"))
        health["checks"]["database"] = "up"
    except Exception:
        health["status"] = "unhealthy"
        health["checks"]["database"] = "down"
    
    # Check Redis (degraded — service works but features limited)
    try:
        await redis.ping()
        health["checks"]["redis"] = "up"
    except RedisError:
        health["status"] = "degraded"  # Not "unhealthy" — service still works
        health["checks"]["redis"] = "down"
    
    status_code = 200 if health["status"] != "unhealthy" else 503
    return JSONResponse(content=health, status_code=status_code)
```

**The status levels:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   HEALTH STATUS LEVELS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "healthy"   — All systems go. Full functionality.              │
│                HTTP 200. All features work.                      │
│                                                                 │
│  "degraded"  — Core works, optional services impaired.          │
│                HTTP 200. Auth works (JWT), but no revocation,   │
│                no rate limiting, no session data, no cache.     │
│                Alert the on-call engineer.                       │
│                                                                 │
│  "unhealthy" — Core services down. Cannot serve requests.       │
│                HTTP 503. Database is unreachable.                │
│                Load balancer should stop routing traffic here.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              SESSION & TOKEN STORAGE — QUICK REFERENCE          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  KEY NAMING:                                                    │
│     refresh:{jti}              Refresh token record (TTL)       │
│     user_sessions:{user_id}    SET of active JTIs              │
│     blocklist:{jti}            Revoked access token (short TTL) │
│     session:{user_id}          HASH of session fields (TTL)    │
│     ratelimit:{id}:{window}    Request counter (TTL)           │
│                                                                 │
│  STORE REFRESH TOKEN:                                           │
│     SETEX refresh:{jti} 604800 {user_data}                     │
│     SADD user_sessions:{user_id} {jti}                         │
│                                                                 │
│  VALIDATE REFRESH TOKEN:                                        │
│     GET refresh:{jti} → exists? valid : rejected                │
│                                                                 │
│  ROTATE TOKEN:                                                  │
│     DEL refresh:{old_jti}                                       │
│     SETEX refresh:{new_jti} 604800 {data}                      │
│     SREM user_sessions:{uid} {old_jti}                         │
│     SADD user_sessions:{uid} {new_jti}                         │
│                                                                 │
│  REVOKE ONE SESSION:                                            │
│     DEL refresh:{jti}                                           │
│     SREM user_sessions:{uid} {jti}                             │
│     SETEX blocklist:{access_jti} {remaining_ttl} "1"           │
│                                                                 │
│  REVOKE ALL SESSIONS:                                           │
│     SMEMBERS user_sessions:{uid} → get all JTIs                │
│     DEL refresh:{jti1} refresh:{jti2} ...                      │
│     DEL user_sessions:{uid}                                     │
│                                                                 │
│  RATE LIMIT (fixed window):                                     │
│     INCR ratelimit:{id}:{window}                                │
│     EXPIRE ratelimit:{id}:{window} {window_seconds}            │
│     if count > limit → 429                                      │
│                                                                 │
│  SESSION DATA:                                                  │
│     HSET session:{uid} field value                              │
│     HGET session:{uid} field                                    │
│     DEL session:{uid}   (logout)                                │
│                                                                 │
│  GRACEFUL DEGRADATION:                                          │
│     Blocklist check fails → skip, log warning                   │
│     Refresh validation fails → reject (safe default)            │
│     Rate limit fails → in-memory fallback                       │
│     Session read fails → use defaults                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  REDIS = YOUR SECURITY DESK                                     │
│                                                                 │
│  It sits between your door scanners (JWT validation)            │
│  and your HR database (PostgreSQL). It answers fast             │
│  questions about the CURRENT STATE of your building:            │
│                                                                 │
│  "Is badge #abc still active?"          → GET refresh:abc       │
│  "Put badge #abc on the stolen list"    → SETEX blocklist:abc   │
│  "Deactivate ALL of Employee 42's       → SMEMBERS + DEL       │
│   badges immediately"                                           │
│  "How many times has this IP tried      → INCR + EXPIRE        │
│   the front door in the last minute?"                           │
│  "What floor is Employee 42 on?"        → HGET session:42      │
│                                                                 │
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   Client     │    │   Redis     │    │ PostgreSQL  │         │
│  │  (visitor)   │    │ (security   │    │ (HR system) │         │
│  │             │    │   desk)     │    │             │         │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘         │
│         │                  │                  │                 │
│         │  badge scan      │                  │                 │
│         ├─────────────────▶│  "active?"       │                 │
│         │                  │ (sub-ms)         │                 │
│         │   ✅ or ❌        │                  │                 │
│         │◀─────────────────┤                  │                 │
│         │                  │                  │                 │
│         │    (if ✅)        │  get full record │                 │
│         ├──────────────────┼─────────────────▶│                 │
│         │                  │                  │ (2-5ms)         │
│         │    user data     │                  │                 │
│         │◀─────────────────┼──────────────────┤                 │
│                                                                 │
│  HYBRID AUTH = best of both worlds:                             │
│  ├─ JWT for fast, stateless signature checks (every request)   │
│  ├─ Redis for fast, stateful control (revocation, sessions)    │
│  └─ PostgreSQL for durable, permanent records (users, roles)   │
│                                                                 │
│  DESIGN FOR FAILURE:                                            │
│  Redis down? JWT still works. You lose revocation temporarily. │
│  PostgreSQL down? Nothing works. That's your real dependency.   │
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
│  WEEK 10 PROJECT: Add Caching Layer                             │
│  └─ You'll implement everything from this lecture alongside     │
│     your cache layer. Token storage and caching coexist in      │
│     the same Redis instance — key namespacing keeps them        │
│     separate.                                                   │
│                                                                 │
│  WEEK 11: Background Jobs & Event-Driven Patterns               │
│  └─ Stale entry cleanup (user_sessions sets) can be handled    │
│     by a periodic Celery Beat task instead of lazy cleanup.     │
│     Token revocation events can trigger notifications           │
│     ("Your session was terminated from another device").        │
│                                                                 │
│  WEEK 12: WebSockets & Real-Time                                │
│  └─ WebSocket connections need authentication too.              │
│     Token validation on connection open uses the same           │
│     TokenService. Session data tracks WebSocket state.          │
│                                                                 │
│  WEEK 12: Rate Limiting Your API                                │
│  └─ You'll use slowapi, which uses the same Redis INCR+EXPIRE  │
│     pattern under the hood. You already understand the          │
│     mechanism — the library just wraps it with decorators.      │
│                                                                 │
│  WEEK 13-14: Capstone (Multi-Tenant SaaS)                       │
│  └─ Multi-tenant sessions: current_org_id in session data.     │
│     Audit logging: token events feed the audit trail.           │
│     All-device logout is a required feature.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```