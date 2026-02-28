# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THREAT FIRST, CODE LAST                                        │
│  ───────────────────────                                        │
│  Students must understand WHY each decision was made before     │
│  they see the implementation. Every line of code exists          │
│  because of a specific attack it defends against.               │
│                                                                 │
│  ARMS RACE NARRATIVE                                            │
│  ───────────────────                                            │
│  We build understanding layer by layer: each "solution"         │
│  reveals a new vulnerability, until we arrive at the            │
│  correct approach. Peel the onion.                              │
│                                                                 │
│  BUILDING SECURITY ANALOGY                                      │
│  ─────────────────────────                                      │
│  Authentication is abstract. We use a building security         │
│  analogy throughout. Every concept maps to something            │
│  tangible you've experienced walking into an office.            │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Pydantic models → request validation for registration/login    │
│  SQLAlchemy → User model with password_hash column              │
│  Custom exceptions → AuthenticationError, DuplicateUserError    │
│  FastAPI dependencies → auth utilities become Depends()         │
│  httpx / external APIs → you just used API keys; now build      │
│    the system that ISSUES credentials                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                  AUTHENTICATION FUNDAMENTALS                    │
│                      (3-3.5 Hour Lecture)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (25 min)                                   │
│  ├─ 1.1 The Unguarded API (Demonstration)                       │
│  ├─ 1.2 Authentication vs Authorization                         │
│  └─ 1.3 The Building Security Analogy                           │
│                                                                 │
│  PART 2: THE PASSWORD PROBLEM (45 min)                          │
│  ├─ 2.1 The Breach Scenario                                     │
│  ├─ 2.2 Plaintext: The Cardinal Sin                             │
│  ├─ 2.3 Why Not Just Encrypt?                                   │
│  ├─ 2.4 Hashing: One-Way Functions                              │
│  ├─ 2.5 The Speed Problem (MD5/SHA Benchmark)                   │
│  └─ 2.6 Rainbow Tables and Salting                              │
│                                                                 │
│  PART 3: MODERN PASSWORD HASHING (40 min)                       │
│  ├─ 3.1 The Properties We Need                                  │
│  ├─ 3.2 Argon2id: The Modern Champion                           │
│  ├─ 3.3 Anatomy of an Argon2id Hash                             │
│  ├─ 3.4 Bcrypt: The Reliable Veteran                            │
│  ├─ 3.5 pwdlib: Python Implementation                           │
│  └─ 3.6 Legacy: passlib                                         │
│                                                                 │
│  PART 4: BUILDING AUTHENTICATION (50 min)                       │
│  ├─ 4.1 The User Model (Connection to Week 6)                   │
│  ├─ 4.2 Password Hashing Utility Module                         │
│  ├─ 4.3 Registration Flow                                       │
│  ├─ 4.4 Login Flow                                              │
│  └─ 4.5 Password Requirements (What NIST Actually Says)         │
│                                                                 │
│  PART 5: COMMON MISTAKES AND SECURITY MINDSET (20 min)          │
│  ├─ 5.1 Error Messages That Leak Information                    │
│  ├─ 5.2 Never Log Passwords                                     │
│  ├─ 5.3 Timing Attacks (Brief)                                  │
│  └─ 5.4 The Security Mindset                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Unguarded API

**Start with a demonstration. Make them feel the danger.**

> "You've built a Task Manager API over the past several weeks. It has endpoints, database persistence, tests, pagination. It's polished. But right now, watch this."

```python
# demo_no_auth.py — Open a terminal and run these with students watching
# Assume our Task Manager from Week 6 is running at localhost:8000

# ---- Terminal 1: Alice creates a private task ----
# curl -X POST http://localhost:8000/api/v1/tasks \
#   -H "Content-Type: application/json" \
#   -d '{"title": "Fire underperforming contractor", "priority": "high"}'
# Response: {"id": 1, "title": "Fire underperforming contractor", ...}

# ---- Terminal 2: A stranger (or the contractor) reads it ----
# curl http://localhost:8000/api/v1/tasks/1
# Response: {"id": 1, "title": "Fire underperforming contractor", ...}

# ---- Terminal 3: The stranger deletes it ----
# curl -X DELETE http://localhost:8000/api/v1/tasks/1
# Response: 204 No Content

# ---- Terminal 1: Alice checks her task ----
# curl http://localhost:8000/api/v1/tasks/1
# Response: 404 Not Found
```

**Now ask the class:**

> "Who created that task? We don't know. Who deleted it? We don't know. Who is making any of these requests? We have no idea. Our API is a building with no front door. Anyone can walk in, read anything, change anything, delete anything."

```
┌─────────────────────────────────────────────────────────────────┐
│               THE UNGUARDED API                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Current state of our API:                                      │
│                                                                 │
│     ANY request ──────▶ ANY endpoint ──────▶ ANY data           │
│                                                                 │
│  No concept of:                                                 │
│  ├─ WHO is making the request  (identity)                       │
│  ├─ WHETHER they should have access  (permission)               │
│  └─ WHETHER they are who they claim to be  (verification)       │
│                                                                 │
│  This is not a toy problem. This is how data breaches happen.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The insight:**

> "Every real API needs to answer two questions before processing any request: 'Who are you?' and 'Are you allowed to do this?' These are two different questions with two different names."

---

## 1.2 Authentication vs Authorization

**Two words that sound similar. They are not the same thing.**

```
┌─────────────────────────────────────────────────────────────────┐
│           AUTHENTICATION vs AUTHORIZATION                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AUTHENTICATION (AuthN)                                         │
│  ──────────────────────                                         │
│  "WHO are you?"                                                 │
│                                                                 │
│  Verifying IDENTITY.                                            │
│  Proving you are who you claim to be.                           │
│                                                                 │
│  Examples:                                                      │
│  • Entering your username and password                          │
│  • Scanning your fingerprint                                    │
│  • Showing your passport at the border                          │
│                                                                 │
│  Result: "This request is from Alice."                          │
│                                                                 │
│                                                                 │
│  AUTHORIZATION (AuthZ)                                          │
│  ────────────────────                                           │
│  "Are you ALLOWED to do this?"                                  │
│                                                                 │
│  Verifying PERMISSION.                                          │
│  Checking what an identified user can access.                   │
│                                                                 │
│  Examples:                                                      │
│  • Can Alice delete OTHER people's tasks? (admin only)          │
│  • Can Alice access the billing page? (owner only)              │
│  • Can a free-tier user use the export feature? (paid only)     │
│                                                                 │
│  Result: "Alice IS allowed to view this task."                  │
│                                                                 │
│                                                                 │
│  ORDER MATTERS:                                                 │
│  ──────────────                                                 │
│  Authentication ALWAYS comes first.                             │
│  You can't check permissions if you don't know who's asking.    │
│                                                                 │
│     Request ──▶ AuthN ("Who?") ──▶ AuthZ ("Allowed?") ──▶ Data │
│                                                                 │
│  TODAY: We build Authentication (Part 4)                        │
│  LECTURE 3: We build Authorization (RBAC)                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 The Building Security Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE BUILDING SECURITY ANALOGY                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NO SECURITY (Our API right now)                                │
│  ───────────────────────────────                                │
│  A building with no front door. No guard. No locks.             │
│  Anyone walks in, takes anything, goes anywhere.                │
│                                                                 │
│                                                                 │
│  AUTHENTICATION (What we build today)                           │
│  ────────────────────────────────────                           │
│  A security guard at the front desk.                            │
│  You show your ID badge. The guard checks:                      │
│  "Is this person registered in our system?"                     │
│  If yes → you're in. If no → door stays locked.                 │
│                                                                 │
│                                                                 │
│  AUTHORIZATION (Lecture 3)                                      │
│  ────────────────────────                                       │
│  Your badge has a color. Red = admin. Blue = employee.          │
│  Red badge opens ALL doors. Blue badge opens YOUR floor only.   │
│  The badge reader checks: "Does this badge open THIS door?"     │
│                                                                 │
│                                                                 │
│  THE CRITICAL QUESTION:                                         │
│  ─────────────────────                                          │
│  How does the guard verify your identity?                       │
│                                                                 │
│  You walk up and say: "I'm Alice, my password is dolphin42."    │
│  The guard needs to CHECK this. But how?                        │
│  What does the guard have stored in the back office?            │
│                                                                 │
│  ──▶ This question is the ENTIRE rest of this lecture. ◀──      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to programming:**

```
Building Security       │  Your FastAPI Backend
────────────────────────│─────────────────────────
Building                │  Your API server
Front door              │  Endpoint protection
Security guard          │  Authentication middleware
ID badge                │  Credentials (username + password)
Badge verification      │  Password hashing + comparison
Back office records     │  Users table in PostgreSQL
Badge color / level     │  User roles (Lecture 3)
Door badge reader       │  Authorization dependency (Lecture 3)
```

---

# PART 2: THE PASSWORD PROBLEM

## 2.1 The Breach Scenario

**Before we write a single line of auth code, we need to think like an attacker.**

> "Assume the worst. Assume your database WILL be breached. Not 'might be.' WILL be. The question is: when that happens, how much damage does the attacker get?"

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE BREACH SCENARIO                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A breach doesn't mean your code was bad.                       │
│  It could be:                                                   │
│  ├─ A misconfigured backup exposed to the internet              │
│  ├─ A disgruntled employee with database access                 │
│  ├─ A SQL injection in a legacy service on the same network     │
│  ├─ A compromised cloud credential                              │
│  └─ A third-party dependency with a vulnerability               │
│                                                                 │
│  Your job is NOT just to prevent breaches.                      │
│  Your job is to ensure that IF a breach happens,                │
│  the attacker gets NOTHING useful.                              │
│                                                                 │
│  Specifically: they should NOT get your users' passwords.       │
│  Because users reuse passwords across sites.                    │
│  One breach becomes EVERY account that user has.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world context:**

> "LinkedIn in 2012: 117 million password hashes leaked. They used unsalted SHA-1. Attackers cracked the majority within days. RockYou in 2009: 32 million passwords stored in plaintext. Adobe in 2013: 153 million passwords stored with reversible encryption. These are not small companies. They had real engineers. They got password storage wrong."

---

## 2.2 Plaintext: The Cardinal Sin

**What if the guard just keeps a notebook?**

> "The simplest approach: store passwords exactly as the user types them."

```python
# ❌ NEVER DO THIS. EVER. FOR ANY REASON.
# This is here to show you the problem, not the solution.

# What a users table looks like with plaintext passwords:
# ┌─────┬──────────┬─────────────────┬──────────────────┐
# │ id  │ username │ email           │ password         │
# ├─────┼──────────┼─────────────────┼──────────────────┤
# │ 1   │ alice    │ [email protected]  │ dolphin42        │
# │ 2   │ bob      │ [email protected]    │ Password123!     │
# │ 3   │ charlie  │ [email protected]│ charlie2024      │
# │ 4   │ diana    │ [email protected]  │ dolphin42        │  ← Same as Alice!
# └─────┴──────────┴─────────────────┴──────────────────┘
```

**What happens when this database is breached:**

```
┌─────────────────────────────────────────────────────────────────┐
│               PLAINTEXT BREACH: GAME OVER                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Attacker gets the database dump.                               │
│  Time to crack ALL passwords: 0 seconds.                        │
│  They're already there. In plain text. Readable.                │
│                                                                 │
│  Now the attacker:                                              │
│  1. Has every user's email + password                           │
│  2. Tries alice's password on Gmail → Works (she reused it)     │
│  3. Tries alice's password on her bank → Works                  │
│  4. Tries bob's password everywhere → Works                     │
│  5. Notices Alice and Diana share "dolphin42"                   │
│     → Knows this is a common password people reuse              │
│                                                                 │
│  Damage: TOTAL. Every user is compromised everywhere.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Back to the analogy:**

> "This is like the security guard keeping a notebook that says 'Alice - dolphin42, Bob - Password123!' Anyone who steals the notebook has every password for every person in the building."

---

## 2.3 Why Not Just Encrypt?

> "A student's first instinct is often: 'Just encrypt the passwords!' Let's examine why that doesn't work."

```
┌─────────────────────────────────────────────────────────────────┐
│                    ENCRYPTION                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Encryption is a TWO-WAY operation.                             │
│  You encrypt with a key. You decrypt with a key.                │
│                                                                 │
│     "dolphin42"  ──encrypt(key)──▶  "x8f2k9m3p..."             │
│     "x8f2k9m3p..." ──decrypt(key)──▶  "dolphin42"              │
│                        ▲                                        │
│                        │                                        │
│                  THE KEY exists somewhere.                       │
│                  If the attacker finds the key,                  │
│                  ALL passwords are immediately exposed.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The problem with encrypted passwords:**

```python
# Encrypted password storage (still bad!)
# ┌─────┬──────────┬──────────────────────────────────────┐
# │ id  │ username │ password_encrypted                   │
# ├─────┼──────────┼──────────────────────────────────────┤
# │ 1   │ alice    │ gAAAAABl2x8f2k9m3pQ7...             │
# │ 2   │ bob      │ gAAAAABl5y3k8n2m1rT4...             │
# └─────┴──────────┴──────────────────────────────────────┘

# The encryption key lives... where?
# In an environment variable? In a config file? In a key vault?
# Wherever it is, the attacker who got your DATABASE
# very likely also got your APPLICATION SERVER.
# And that's where the key is.

# decrypt(key, "gAAAAABl2x8f2k9m3pQ7...") → "dolphin42"
# Now we're back to plaintext.
```

**The fundamental insight:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  The question is:                                               │
│  Do you EVER need to know the user's original password?         │
│                                                                 │
│  At registration: User sends "dolphin42" → You store something  │
│  At login:        User sends "dolphin42" → You check something  │
│                                                                 │
│  You never need to RECOVER the original password.               │
│  You only need to CHECK: "Is what they just typed               │
│  the same as what they typed at registration?"                  │
│                                                                 │
│  You need VERIFICATION, not RECOVERY.                           │
│  This is the key insight that leads us to hashing.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Back to the analogy:**

> "You don't need the guard to remember your face. You need the guard to look at you and confirm: 'Yes, this person matches the record on file.' The guard doesn't need to be able to reconstruct your face from the record. They just need to verify a match."

---

## 2.4 Hashing: One-Way Functions

**Hashing is a ONE-WAY operation. You cannot reverse it.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   HASHING vs ENCRYPTION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ENCRYPTION (Two-way):                                          │
│                                                                 │
│     "dolphin42"  ───encrypt(key)───▶  "x8f2k9m3p..."           │
│     "x8f2k9m3p..." ───decrypt(key)───▶  "dolphin42"            │
│                                                                 │
│     Reversible. Key exists. Key can be stolen.                  │
│                                                                 │
│                                                                 │
│  HASHING (One-way):                                             │
│                                                                 │
│     "dolphin42"  ───hash()───▶  "a7f3b2c1d4e5..."              │
│     "a7f3b2c1d4e5..." ───???───▶  IMPOSSIBLE                   │
│                                                                 │
│     Irreversible. No key. Nothing to steal.                     │
│     You can go forward, but you can NEVER go back.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Properties of a hash function:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 HASH FUNCTION PROPERTIES                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. DETERMINISTIC                                               │
│     Same input ALWAYS produces same output.                     │
│     hash("dolphin42") → "a7f3b2..." every single time          │
│                                                                 │
│  2. ONE-WAY (Pre-image resistance)                              │
│     Given "a7f3b2...", you cannot find "dolphin42"              │
│     There is NO reverse function. None.                         │
│                                                                 │
│  3. FIXED-SIZE OUTPUT                                           │
│     Whether input is 3 characters or 3 million characters,      │
│     the output is always the same length.                       │
│     MD5 → always 32 hex chars                                   │
│     SHA-256 → always 64 hex chars                               │
│                                                                 │
│  4. AVALANCHE EFFECT                                            │
│     Tiny change in input → completely different output.          │
│     hash("dolphin42") → "a7f3b2c1..."                          │
│     hash("dolphin43") → "e91d8f7a..."  ← Totally different     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How verification works with hashing:**

```
┌─────────────────────────────────────────────────────────────────┐
│              HASH-BASED VERIFICATION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REGISTRATION:                                                  │
│  ─────────────                                                  │
│  User: "My password is dolphin42"                               │
│  Server: hash("dolphin42") → "a7f3b2c1d4e5..."                 │
│  Server: store "a7f3b2c1d4e5..." in database                   │
│                                                                 │
│  LOGIN:                                                         │
│  ──────                                                         │
│  User: "My password is dolphin42"                               │
│  Server: hash("dolphin42") → "a7f3b2c1d4e5..."                 │
│  Server: compare with stored "a7f3b2c1d4e5..."                  │
│  Server: MATCH! → Login success                                 │
│                                                                 │
│  WRONG PASSWORD:                                                │
│  ───────────────                                                │
│  User: "My password is dolphin43"                               │
│  Server: hash("dolphin43") → "e91d8f7a0b3c..."                 │
│  Server: compare with stored "a7f3b2c1d4e5..."                  │
│  Server: NO MATCH → Login failed                                │
│                                                                 │
│  ✅ Server never stores the actual password.                    │
│  ✅ Server never needs to "decrypt" anything.                   │
│  ✅ If database is stolen, attacker gets hashes, not passwords. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Show it with Python (they know hashlib from the standard library):**

```python
import hashlib

# Hashing is straightforward
password = "dolphin42"
hashed = hashlib.sha256(password.encode()).hexdigest()
print(hashed)
# "b8e9a3f7c2d1e4a6b5c8d7e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0"

# Same input → same output (deterministic)
hashed_again = hashlib.sha256(password.encode()).hexdigest()
print(hashed == hashed_again)  # True

# Tiny change → completely different output (avalanche effect)
hashed_different = hashlib.sha256("dolphin43".encode()).hexdigest()
print(hashed == hashed_different)  # False — totally different string
```

> "This looks promising. Store the hash, not the password. If the database leaks, the attacker only gets hashes. They can't reverse them. Problem solved, right?"

**Pause. Ask the class.**

> "Can anyone think of a way an attacker could still figure out the original passwords from these hashes?"

---

## 2.5 The Speed Problem (MD5/SHA Benchmark)

**The first crack in our solution: general-purpose hash functions are TOO FAST.**

**Run this live with students watching:**

```python
# demo_hash_speed.py — Run this live.
import hashlib
import time

password = "dolphin42"

# --- MD5 benchmark ---
start = time.time()
for _ in range(1_000_000):
    hashlib.md5(password.encode()).hexdigest()
elapsed = time.time() - start
print(f"MD5:    {1_000_000 / elapsed:>12,.0f} hashes/sec on YOUR laptop")

# --- SHA-256 benchmark ---
start = time.time()
for _ in range(1_000_000):
    hashlib.sha256(password.encode()).hexdigest()
elapsed = time.time() - start
print(f"SHA-256:{1_000_000 / elapsed:>12,.0f} hashes/sec on YOUR laptop")
```

**Typical output:**

```
MD5:     3,200,000 hashes/sec on YOUR laptop
SHA-256: 1,800,000 hashes/sec on YOUR laptop
```

**Now drop this bomb:**

> "That's on YOUR laptop. A single consumer CPU. An attacker doesn't use a laptop. They use GPUs. A single modern GPU can compute tens of BILLIONS of MD5 hashes per second. Not millions. BILLIONS."

```
┌─────────────────────────────────────────────────────────────────┐
│                THE SPEED PROBLEM                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Hashing speed on different hardware:                           │
│                                                                 │
│  YOUR LAPTOP (single CPU):                                      │
│  ├─ MD5:     ~3,000,000 /sec                                    │
│  └─ SHA-256: ~1,800,000 /sec                                    │
│                                                                 │
│  ATTACKER'S GPU (e.g., RTX 4090):                               │
│  ├─ MD5:     ~160,000,000,000 /sec  (164 billion)               │
│  └─ SHA-256: ~22,000,000,000 /sec   (22 billion)                │
│                                                                 │
│                                                                 │
│  How fast can they brute-force passwords?                       │
│  ─────────────────────────────────────────                      │
│  All 8-character lowercase passwords (26^8 = 208 billion):      │
│                                                                 │
│  MD5 on GPU:     208B ÷ 164B/sec ≈ 1.3 seconds                 │
│  SHA-256 on GPU: 208B ÷ 22B/sec  ≈ 9.5 seconds                 │
│                                                                 │
│  Every. Single. 8-character. Lowercase. Password.               │
│  Cracked in under 10 seconds.                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "MD5 and SHA-256 were designed to be fast. They're used for file integrity checks, digital signatures, data deduplication — tasks where speed is a FEATURE. For passwords, speed is the ENEMY. The faster you can hash, the faster an attacker can guess."

**The attack is simple:**

```
Brute-Force Attack:

  for every possible password in ["a", "b", ..., "zzzzzzzz"]:
      if hash(guess) == stolen_hash:
          print(f"Password is: {guess}")
          break

  At 164 billion guesses/sec with MD5, this is fast.
```

---

## 2.6 Rainbow Tables and Salting

**The second crack: precomputed lookup tables.**

> "Brute force isn't even the fastest attack. What if you precompute all the hashes?"

```
┌─────────────────────────────────────────────────────────────────┐
│                  RAINBOW TABLES                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A rainbow table is a PRECOMPUTED database of:                  │
│  password → hash                                                │
│                                                                 │
│  ┌──────────────────┬──────────────────────────────────────┐    │
│  │ password         │ MD5 hash                             │    │
│  ├──────────────────┼──────────────────────────────────────┤    │
│  │ password         │ 5f4dcc3b5aa765d61d8327deb882cf99     │    │
│  │ 123456           │ e10adc3949ba59abbe56e057f20f883e     │    │
│  │ dolphin42        │ a7f3b2c1d4e5...                      │    │
│  │ qwerty           │ d8578edf8458ce06fbc5bb76a58c5ca4     │    │
│  │ ... millions     │ ...                                  │    │
│  └──────────────────┴──────────────────────────────────────┘    │
│                                                                 │
│  Attack: Stolen hash → look up in table → password found        │
│  Time: MILLISECONDS per password (just a database lookup)       │
│                                                                 │
│  Rainbow tables for all 8-char alphanumeric MD5 hashes:         │
│  ~1 TB. Downloadable for free.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The fix: SALTING.**

> "A salt is a random string that you add to the password BEFORE hashing. Each user gets a DIFFERENT salt."

```
┌─────────────────────────────────────────────────────────────────┐
│                    SALTING                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT SALT:                                                  │
│  ─────────────                                                  │
│  hash("dolphin42")  → "a7f3b2c1..."                            │
│  hash("dolphin42")  → "a7f3b2c1..."  ← SAME hash!             │
│                                                                 │
│  Alice and Diana both use "dolphin42".                          │
│  Attacker sees identical hashes → knows they share a password.  │
│  Crack one → crack both. Rainbow table cracks ALL at once.      │
│                                                                 │
│                                                                 │
│  WITH SALT:                                                     │
│  ──────────                                                     │
│  Alice's salt:  "x9k2m"  (randomly generated at registration)  │
│  Diana's salt:  "p4j7n"  (different random value)              │
│                                                                 │
│  hash("x9k2m" + "dolphin42")  → "f82d1a9c..."                 │
│  hash("p4j7n" + "dolphin42")  → "3b7e0c5d..."  ← DIFFERENT!   │
│                                                                 │
│  Same password, different hashes.                               │
│  Rainbow tables are now USELESS because they were computed      │
│  without the salt. Every salt creates a unique hash space.      │
│                                                                 │
│  The salt is stored alongside the hash (it's NOT a secret).     │
│  Its purpose is to make each hash UNIQUE, not to be hidden.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Demonstrate with code:**

```python
import hashlib
import os

# Generate a random salt (16 bytes = 128 bits)
salt = os.urandom(16)
print(f"Salt: {salt.hex()}")  # e.g., "a3f8c2e1b5d4..."

# Hash with salt
password = "dolphin42"
salted = salt + password.encode()
hashed = hashlib.sha256(salted).hexdigest()
print(f"Hash: {hashed}")

# Same password, different salt → different hash
salt2 = os.urandom(16)
salted2 = salt2 + password.encode()
hashed2 = hashlib.sha256(salted2).hexdigest()
print(f"Hash2: {hashed2}")

print(f"Same password, same hash? {hashed == hashed2}")  # False!
```

> "Salting kills rainbow tables. But it does NOT slow down brute force. An attacker with the stolen salt can still try billions of guesses per second — they just have to include the salt in each guess. We solved one problem. The speed problem remains."

```
┌─────────────────────────────────────────────────────────────────┐
│            SCORECARD: WHERE WE STAND                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                        Plaintext  Encrypted  MD5  Salted MD5    │
│  ──────────────────────────────────────────────────────────────  │
│  Breach = passwords?    YES        YES*       NO     NO         │
│  Rainbow table attack?  N/A        N/A        YES    NO ✅      │
│  Brute force attack?    N/A        N/A        FAST   FAST ❌    │
│                                                                 │
│  * Encrypted = YES if attacker also gets the encryption key     │
│                                                                 │
│  We need something that is SLOW to compute.                     │
│  Not accidentally slow. DELIBERATELY slow.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: MODERN PASSWORD HASHING

## 3.1 The Properties We Need

> "Now we know what we need. Let's write the wishlist for the perfect password hashing algorithm."

```
┌─────────────────────────────────────────────────────────────────┐
│          IDEAL PASSWORD HASHING ALGORITHM                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ ONE-WAY (irreversible)                                      │
│     Cannot recover password from hash.                          │
│                                                                 │
│  ✅ SALTED (unique per user)                                    │
│     Same password → different hash for each user.               │
│     Kills rainbow tables.                                       │
│                                                                 │
│  ✅ DELIBERATELY SLOW (key stretching)                          │
│     Takes significant time per hash (e.g., 100ms+).             │
│     Attacker can only try ~10 guesses/sec instead of billions.  │
│     Has a configurable "cost factor" that increases over time.  │
│                                                                 │
│  ✅ MEMORY-HARD (defeats GPUs)                                  │
│     Requires large amounts of RAM per hash computation.         │
│     GPUs have thousands of cores but limited RAM per core.      │
│     If each hash needs 64MB of RAM, a GPU with 24GB VRAM       │
│     can only run ~375 parallel attempts instead of thousands.   │
│                                                                 │
│  ✅ CONFIGURABLE (future-proof)                                 │
│     Parameters (time, memory, parallelism) can be tuned         │
│     upward as hardware gets faster, without changing code.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why "deliberately slow" matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE MATH OF SLOW HASHING                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Attacker tries to crack an 8-char password (lowercase+digits)  │
│  Total combinations: 36^8 ≈ 2.8 trillion                       │
│                                                                 │
│  With MD5 (fast):                                               │
│  ├─ GPU speed: ~160 billion/sec                                 │
│  └─ Time to exhaust: ~17 seconds                                │
│                                                                 │
│  With Argon2id (slow + memory-hard):                            │
│  ├─ Effective speed: ~10/sec (memory bottleneck on GPU)         │
│  └─ Time to exhaust: ~8,900 YEARS                               │
│                                                                 │
│  Same password. Same attacker. Same hardware.                   │
│  17 seconds vs 8,900 years.                                     │
│  That's the difference a proper algorithm makes.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Argon2id: The Modern Champion

> "In 2013, the cryptography community recognized that the world needed a better password hashing standard. They organized the Password Hashing Competition — an open, multi-year international contest to find the best algorithm."

**Argon2id is the answer.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     ARGON2ID                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Winner of the Password Hashing Competition (July 2015).        │
│  Recommended by OWASP and RFC 9106.                             │
│                                                                 │
│  Designed by:                                                   │
│  Alex Biryukov, Daniel Dinu, Dmitry Khovratovich               │
│  University of Luxembourg                                       │
│                                                                 │
│                                                                 │
│  THREE CONFIGURABLE COSTS:                                      │
│  ─────────────────────────                                      │
│                                                                 │
│  1. TIME COST (t) — Number of iterations                        │
│     More iterations = slower to compute.                        │
│     Like making the guard check your ID 3 times.                │
│                                                                 │
│  2. MEMORY COST (m) — RAM required per hash                     │
│     e.g., 64 MB per hash computation.                           │
│     GPUs have thousands of cores, but each core has little      │
│     memory. If each hash needs 64MB, parallelism collapses.     │
│     THIS is what defeats GPU attacks.                           │
│                                                                 │
│  3. PARALLELISM (p) — Number of threads                         │
│     How many CPU threads the hash can use.                      │
│     Prevents attackers from gaining unfair advantage             │
│     from hardware parallelism.                                  │
│                                                                 │
│                                                                 │
│  WHY "id" IN ARGON2ID?                                          │
│  ─────────────────────                                          │
│  Three variants exist:                                          │
│  ├─ Argon2d  — Resists GPU attacks, vulnerable to side-channel  │
│  ├─ Argon2i  — Resists side-channel, slower vs GPU attacks      │
│  └─ Argon2id — HYBRID. Best of both. The recommended variant.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now demonstrate the speed difference:**

```python
# demo_argon2_speed.py — Run this live. The contrast is devastating.
import hashlib
import time

password = "dolphin42"

# --- MD5: blazingly fast (BAD for passwords) ---
start = time.time()
for _ in range(1_000_000):
    hashlib.md5(password.encode()).hexdigest()
md5_elapsed = time.time() - start
md5_per_sec = 1_000_000 / md5_elapsed

# --- SHA-256: still very fast (BAD for passwords) ---
start = time.time()
for _ in range(1_000_000):
    hashlib.sha256(password.encode()).hexdigest()
sha_elapsed = time.time() - start
sha_per_sec = 1_000_000 / sha_elapsed

# --- Argon2id: deliberately slow (GOOD for passwords) ---
from pwdlib import PasswordHash
hasher = PasswordHash.recommended()

start = time.time()
for _ in range(10):
    hasher.hash(password)
argon_elapsed = time.time() - start
argon_per_sec = 10 / argon_elapsed

print(f"MD5:      {md5_per_sec:>12,.0f} hashes/sec")
print(f"SHA-256:  {sha_per_sec:>12,.0f} hashes/sec")
print(f"Argon2id: {argon_per_sec:>12.1f} hashes/sec")  # Single digits!
```

**Typical output (let the class absorb this):**

```
MD5:       3,200,000 hashes/sec
SHA-256:   1,800,000 hashes/sec
Argon2id:        3.2 hashes/sec
```

> "Read those numbers again. MD5: three million per second. Argon2id: three per second. That's a million-fold difference. On YOUR laptop. On an attacker's GPU, that ratio is even more extreme because Argon2id's memory requirements prevent GPU parallelism."

---

## 3.3 Anatomy of an Argon2id Hash

**What does Argon2id actually store in your database?**

```python
from pwdlib import PasswordHash

hasher = PasswordHash.recommended()
stored_hash = hasher.hash("dolphin42")
print(stored_hash)
# $argon2id$v=19$m=65536,t=3,p=4$a3JmN2s4bTJuOHE1dDly$K9pMi...x8Y
```

**Dissect every part:**

```
┌─────────────────────────────────────────────────────────────────┐
│              ANATOMY OF AN ARGON2ID HASH                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  $argon2id$v=19$m=65536,t=3,p=4$a3JmN2s4bT...$K9pMi...x8Y     │
│  │        │    │               │              │                 │
│  │        │    │               │              └─ Hash output    │
│  │        │    │               │                 (base64)       │
│  │        │    │               │                                │
│  │        │    │               └─ Salt (base64, auto-generated) │
│  │        │    │                                                │
│  │        │    └─ Parameters:                                   │
│  │        │       m=65536  Memory: 65536 KiB (64 MB)            │
│  │        │       t=3      Time: 3 iterations                   │
│  │        │       p=4      Parallelism: 4 threads               │
│  │        │                                                     │
│  │        └─ Version: 19 (0x13)                                 │
│  │                                                              │
│  └─ Algorithm identifier: argon2id                              │
│                                                                 │
│                                                                 │
│  KEY INSIGHT:                                                   │
│  ────────────                                                   │
│  The SALT is embedded IN the hash string!                       │
│  You do NOT store the salt separately.                          │
│  The verify() function extracts the salt automatically.         │
│                                                                 │
│  The PARAMETERS are also embedded!                              │
│  So if you change parameters later, old hashes still verify     │
│  because they carry their own parameters.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This is elegant. One string in one database column contains everything needed to verify a password: the algorithm, the version, the tuning parameters, the salt, and the hash output. No extra columns. No separate salt table."

---

## 3.4 Bcrypt: The Reliable Veteran

> "You'll encounter bcrypt everywhere in existing codebases. It was the gold standard before Argon2."

```
┌─────────────────────────────────────────────────────────────────┐
│                     BCRYPT                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Introduced: 1999 (based on Blowfish cipher)                    │
│  Status: Still secure. Widely deployed.                         │
│                                                                 │
│  ✅ Slow (configurable cost factor, "work factor")              │
│  ✅ Salted automatically (128-bit salt embedded in hash)        │
│  ✅ Time-tested (25+ years of cryptanalysis)                    │
│                                                                 │
│  ❌ NOT memory-hard (vulnerable to GPU/ASIC acceleration)       │
│  ❌ 72-byte password limit (silently truncates longer input)    │
│                                                                 │
│  Hash format:                                                   │
│  $2b$12$LJ3m4ks92jfK8snY7dM3qe.KLn2QxPq3GkR7sN2p5aBwK9xMz     │
│   │   │  │                      │                               │
│   │   │  └─ salt (22 chars)     └─ hash (31 chars)              │
│   │   └─ cost factor (2^12 = 4096 iterations)                   │
│   └─ version (2b)                                               │
│                                                                 │
│                                                                 │
│  WHEN YOU'LL SEE IT:                                            │
│  ├─ Older FastAPI/Django/Flask projects                         │
│  ├─ Ruby on Rails applications (bcrypt is the default)          │
│  ├─ Node.js backends (bcrypt.js is extremely popular)           │
│  └─ Any codebase started before ~2020                           │
│                                                                 │
│  OUR CHOICE: Argon2id                                           │
│  ├─ It's newer, stronger (memory-hard), and OWASP-recommended  │
│  └─ But bcrypt is NOT broken. Don't panic if you inherit it.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 pwdlib: Python Implementation

**Install it (using uv from Week 1 Lecture 4):**

```bash
uv add "pwdlib[argon2]"
```

**The core API — three methods you need:**

```python
from pwdlib import PasswordHash

# Create a hasher with recommended settings (Argon2id by default)
password_hasher = PasswordHash.recommended()

# 1. HASH a password (at registration)
hashed = password_hasher.hash("dolphin42")
print(hashed)
# $argon2id$v=19$m=65536,t=3,p=4$...salt...$...hash...

# 2. VERIFY a password (at login)
is_valid = password_hasher.verify("dolphin42", hashed)
print(is_valid)  # True

is_valid = password_hasher.verify("wrong_password", hashed)
print(is_valid)  # False

# 3. VERIFY AND UPDATE (for algorithm migration)
valid, updated_hash = password_hasher.verify_and_update("dolphin42", hashed)
# valid = True/False
# updated_hash = new hash string if algorithm changed, None if current
```

**Why `verify_and_update` exists:**

```
┌─────────────────────────────────────────────────────────────────┐
│              ALGORITHM MIGRATION                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Scenario: Your app originally used bcrypt. You upgrade to      │
│  Argon2id. But existing users still have bcrypt hashes in DB.   │
│                                                                 │
│  verify_and_update() solves this:                               │
│  1. Verifies the password against the OLD hash (bcrypt)         │
│  2. If valid, re-hashes with the CURRENT algorithm (Argon2id)   │
│  3. Returns the new hash so you can update the database         │
│                                                                 │
│  Users seamlessly migrate to the stronger algorithm             │
│  the next time they log in. No forced password resets needed.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Custom configuration (when you need control):**

```python
from pwdlib import PasswordHash
from pwdlib.hashers.argon2 import Argon2Hasher
from pwdlib.hashers.bcrypt import BcryptHasher

# Custom configuration: Argon2id primary, bcrypt for legacy support
password_hasher = PasswordHash((
    Argon2Hasher(),   # New hashes use this (first in tuple = current)
    BcryptHasher(),   # Can still verify old bcrypt hashes
))

# New passwords → Argon2id
# Old bcrypt hashes → still verifiable
# verify_and_update() → upgrades bcrypt → Argon2id on next login
```

---

## 3.6 Legacy: passlib

> "You will encounter passlib in existing codebases. You need to know why it's there and why we don't use it."

```
┌─────────────────────────────────────────────────────────────────┐
│                    PASSLIB (LEGACY)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  passlib was the standard Python password hashing library        │
│  for over a decade. It supported dozens of algorithms.          │
│                                                                 │
│  STATUS: Unmaintained. Broken on Python 3.13+.                  │
│                                                                 │
│  What it looked like:                                           │
│  ─────────────────────                                          │
│  from passlib.context import CryptContext                       │
│  pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")│
│  pwd_context.hash("dolphin42")                                  │
│  pwd_context.verify("dolphin42", stored_hash)                   │
│                                                                 │
│  What to do if you see it:                                      │
│  ─────────────────────────                                      │
│  1. Understand the code (it works the same conceptually)        │
│  2. Plan migration to pwdlib when upgrading Python              │
│  3. Use pwdlib's multi-algorithm support for gradual migration  │
│                                                                 │
│  DO NOT use passlib in new projects.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: BUILDING AUTHENTICATION

## 4.1 The User Model (Connection to Week 6)

> "Using SQLAlchemy 2.0 models (Week 6), let's define our User. Notice: the column is `password_hash`, never `password`."

```python
# app/models/user.py
import uuid
from datetime import datetime

from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column

from app.database import Base


class User(Base):
    __tablename__ = "users"

    id: Mapped[uuid.UUID] = mapped_column(
        primary_key=True,
        default=uuid.uuid4,
    )
    email: Mapped[str] = mapped_column(
        String(255),
        unique=True,
        index=True,
    )
    username: Mapped[str] = mapped_column(
        String(50),
        unique=True,
        index=True,
    )
    password_hash: Mapped[str] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

**Notice the deliberate naming:**

```
┌─────────────────────────────────────────────────────────────────┐
│                COLUMN NAMING MATTERS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌  password         — Implies plaintext. Misleading.          │
│  ❌  encrypted_pwd    — Implies encryption. Wrong concept.      │
│  ❌  pwd              — Vague, unprofessional.                   │
│  ✅  password_hash    — Communicates exactly what's stored.      │
│  ✅  hashed_password  — Also acceptable.                        │
│                                                                 │
│  The column name is documentation. It tells every developer     │
│  who reads this schema: "This is a hash, not a password."       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Don't forget the migration (Alembic, Week 6):**

```bash
alembic revision --autogenerate -m "add_users_table"
alembic upgrade head
```

---

## 4.2 Password Hashing Utility Module

> "We wrap pwdlib in our own utility module. This gives us a single place to change hashing configuration, and keeps our route handlers clean."

```python
# app/core/security.py
from pwdlib import PasswordHash

# Single instance, shared across the application.
# PasswordHash.recommended() defaults to Argon2id.
_password_hasher = PasswordHash.recommended()


def hash_password(plain_password: str) -> str:
    """Hash a plaintext password for storage.

    Called during user registration.
    Returns an Argon2id hash string containing
    the algorithm, parameters, salt, and hash.
    """
    return _password_hasher.hash(plain_password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify a plaintext password against a stored hash.

    Called during login.
    Returns True if the password matches, False otherwise.
    """
    return _password_hasher.verify(plain_password, hashed_password)
```

**Why wrap it?**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY THE WRAPPER?                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. SINGLE POINT OF CHANGE                                      │
│     If you migrate from pwdlib to another library,              │
│     you change ONE file, not every route handler.               │
│                                                                 │
│  2. CLEAR FUNCTION SIGNATURES                                   │
│     hash_password(plain) and verify_password(plain, hashed)     │
│     are self-documenting. The underlying library's argument     │
│     order doesn't matter.                                       │
│                                                                 │
│  3. TESTABILITY                                                 │
│     You can mock app.core.security.hash_password in tests.      │
│     Much cleaner than mocking the library directly.             │
│                                                                 │
│  4. CONFIGURATION ISOLATION                                     │
│     If you add bcrypt fallback support, it stays in this file.  │
│     Route handlers never know or care about hash algorithms.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Registration Flow

> "Now let's build the registration endpoint. This is where security decisions compound."

**First, the Pydantic schemas (Week 3 connection):**

```python
# app/schemas/user.py
import uuid
from datetime import datetime

from pydantic import BaseModel, ConfigDict, EmailStr, Field, field_validator


class UserCreate(BaseModel):
    """Schema for user registration request."""

    email: EmailStr
    username: str = Field(
        min_length=3,
        max_length=50,
        pattern=r"^[a-zA-Z0-9_]+$",
    )
    password: str = Field(min_length=8, max_length=128)

    @field_validator("password")
    @classmethod
    def password_must_not_contain_username(cls, v: str, info) -> str:
        """Prevent passwords that contain the username."""
        if "username" in info.data and info.data["username"].lower() in v.lower():
            raise ValueError("Password must not contain your username")
        return v


class UserResponse(BaseModel):
    """Schema for user data in responses. NEVER includes password_hash."""

    id: uuid.UUID
    email: str
    username: str
    is_active: bool
    created_at: datetime

    model_config = ConfigDict(from_attributes=True)
```

**Critical: UserResponse NEVER includes the hash.**

```
┌─────────────────────────────────────────────────────────────────┐
│           REQUEST vs RESPONSE MODELS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  UserCreate (request):                                          │
│  ├─ email      ✅                                               │
│  ├─ username   ✅                                               │
│  └─ password   ✅ (plaintext — only lives in memory briefly)    │
│                                                                 │
│  UserResponse (response):                                       │
│  ├─ id         ✅                                               │
│  ├─ email      ✅                                               │
│  ├─ username   ✅                                               │
│  ├─ is_active  ✅                                               │
│  ├─ created_at ✅                                               │
│  ├─ password   ❌ NEVER                                         │
│  └─ password_hash ❌ NEVER                                      │
│                                                                 │
│  The password_hash exists in your database and ONLY in your     │
│  database. It never appears in any API response. Ever.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The registration endpoint:**

```python
# app/routers/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.security import hash_password
from app.database import get_session
from app.models.user import User
from app.schemas.user import UserCreate, UserResponse

router = APIRouter(prefix="/auth", tags=["Authentication"])


@router.post(
    "/register",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
)
async def register(
    user_data: UserCreate,
    session: AsyncSession = Depends(get_session),
) -> User:
    """Register a new user account."""

    # --- Step 1: Check for duplicate email ---
    result = await session.execute(
        select(User).where(User.email == user_data.email)
    )
    if result.scalar_one_or_none():
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="Email already registered",
        )

    # --- Step 2: Check for duplicate username ---
    result = await session.execute(
        select(User).where(User.username == user_data.username)
    )
    if result.scalar_one_or_none():
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="Username already taken",
        )

    # --- Step 3: Hash the password and create user ---
    user = User(
        email=user_data.email,
        username=user_data.username,
        password_hash=hash_password(user_data.password),
        #               ▲
        #               │
        #  "dolphin42" goes in, "$argon2id$v=19$..." comes out.
        #  The plaintext password is NEVER stored.
        #  After this line, the only copy of "dolphin42" is in
        #  the UserCreate object, which Python will garbage-collect.
    )
    session.add(user)
    await session.commit()
    await session.refresh(user)

    return user
```

**Visualize the registration flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  REGISTRATION FLOW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client                   Server                   Database     │
│  ──────                   ──────                   ────────     │
│    │                        │                        │          │
│    │  POST /auth/register   │                        │          │
│    │  {email, username,     │                        │          │
│    │   password}            │                        │          │
│    │ ─────────────────────▶ │                        │          │
│    │                        │                        │          │
│    │                  Pydantic validates              │          │
│    │                  (email format, length,          │          │
│    │                   pattern, custom rules)         │          │
│    │                        │                        │          │
│    │                  Check duplicate email ────────▶ │          │
│    │                        │ ◀──── No match         │          │
│    │                  Check duplicate username ─────▶ │          │
│    │                        │ ◀──── No match         │          │
│    │                        │                        │          │
│    │                  hash_password("dolphin42")      │          │
│    │                        │                        │          │
│    │                  → "$argon2id$v=19$..."          │          │
│    │                        │                        │          │
│    │                  INSERT user ──────────────────▶ │          │
│    │                        │              (stores hash, not     │
│    │                        │               the password)       │
│    │                        │                        │          │
│    │  201 Created           │                        │          │
│    │  {id, email, username} │                        │          │
│    │ ◀───────────────────── │                        │          │
│    │                        │                        │          │
│    │  (no password in       │                        │          │
│    │   response. EVER.)     │                        │          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Login Flow

**Now the verification side. This is where attackers probe.**

**First, the login schema:**

```python
# app/schemas/auth.py
from pydantic import BaseModel


class LoginRequest(BaseModel):
    """Schema for login request."""

    username: str
    password: str


class LoginResponse(BaseModel):
    """Schema for login response.
    In Lecture 2, this will return a JWT token.
    For now, we confirm the mechanism works.
    """

    message: str
    # access_token: str    ← Coming in Lecture 2
    # token_type: str      ← Coming in Lecture 2
```

**The login endpoint:**

```python
@router.post("/login", response_model=LoginResponse)
async def login(
    credentials: LoginRequest,
    session: AsyncSession = Depends(get_session),
) -> dict:
    """Authenticate a user and return confirmation."""

    # --- Step 1: Find the user by username ---
    result = await session.execute(
        select(User).where(User.username == credentials.username)
    )
    user = result.scalar_one_or_none()

    # --- Step 2: Verify password ---
    # CRITICAL: Use the SAME error for "user not found" AND "wrong password"
    if not user or not verify_password(credentials.password, user.password_hash):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",    # ← Deliberately vague!
        )

    # --- Step 3: Check if account is active ---
    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Account is disabled",
        )

    # In Lecture 2, we generate and return a JWT token here
    return {"message": "Login successful"}
```

**THE most important line in this function:**

```python
    if not user or not verify_password(credentials.password, user.password_hash):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",
        )
```

**Why this is written the way it is:**

```
┌─────────────────────────────────────────────────────────────────┐
│            WHY ONE ERROR FOR TWO CASES?                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Case 1: Username doesn't exist (user is None)                  │
│  Case 2: Username exists, password is wrong                     │
│                                                                 │
│  ❌ WRONG: Different error messages                             │
│     "User not found"     → Attacker now knows this user         │
│     "Wrong password"        does NOT exist.                     │
│                             Tries another username.             │
│                             Eventually confirms which           │
│                             usernames ARE valid.                │
│                             Then brute-forces passwords         │
│                             for valid usernames only.           │
│                                                                 │
│  ✅ CORRECT: Same error message                                 │
│     "Invalid credentials" → Attacker doesn't know if the       │
│     "Invalid credentials"    username was wrong, the password   │
│                              was wrong, or both.                │
│                              No information gained.             │
│                                                                 │
│  This is called PREVENTING USER ENUMERATION.                    │
│  Section 5.1 covers this in more detail.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visualize the login flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      LOGIN FLOW                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client                   Server                   Database     │
│  ──────                   ──────                   ────────     │
│    │                        │                        │          │
│    │  POST /auth/login      │                        │          │
│    │  {username, password}  │                        │          │
│    │ ─────────────────────▶ │                        │          │
│    │                        │                        │          │
│    │                  SELECT user WHERE               │          │
│    │                  username = ? ─────────────────▶ │          │
│    │                        │ ◀──── User row          │          │
│    │                        │       (with hash)       │          │
│    │                        │                        │          │
│    │                  verify_password(                │          │
│    │                    "dolphin42",                  │          │
│    │                    "$argon2id$v=19$..."          │          │
│    │                  )                              │          │
│    │                        │                        │          │
│    │                  Argon2id:                       │          │
│    │                  1. Extract salt from hash       │          │
│    │                  2. Extract parameters (m,t,p)   │          │
│    │                  3. Hash "dolphin42" with         │          │
│    │                     same salt + parameters       │          │
│    │                  4. Compare result with           │          │
│    │                     stored hash output           │          │
│    │                  5. Match? → True                │          │
│    │                        │                        │          │
│    │  200 OK                │                        │          │
│    │  {"message": "Login    │                        │          │
│    │   successful"}         │                        │          │
│    │ ◀───────────────────── │                        │          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 Password Requirements (What NIST Actually Says)

> "Your instinct might be: 'Require uppercase, lowercase, number, and symbol.' Turns out, the U.S. National Institute of Standards and Technology disagrees."

**NIST Special Publication 800-63B — Digital Identity Guidelines:**

```
┌─────────────────────────────────────────────────────────────────┐
│              NIST PASSWORD GUIDELINES (SP 800-63B)              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ DO:                                                         │
│  ├─ Require minimum 8 characters                                │
│  ├─ Support maximum of at least 64 characters                   │
│  ├─ Allow ALL printable ASCII and Unicode characters             │
│  │  (spaces, emojis, special characters — all of them)          │
│  ├─ Check against lists of breached/common passwords            │
│  │  (e.g., "password123", "qwerty", "letmein")                  │
│  └─ Hash with a memory-hard function (Argon2id)                 │
│                                                                 │
│  ❌ DO NOT:                                                     │
│  ├─ Require composition rules (uppercase + number + symbol)     │
│  │  → Users write "P@ssw0rd!" which is in every wordlist        │
│  ├─ Force periodic password changes                             │
│  │  → Users increment: "Password1" → "Password2" → "Password3" │
│  ├─ Use password hints                                          │
│  │  → Leaks information about the password                      │
│  └─ Use knowledge-based questions ("mother's maiden name")      │
│     → Answers are often publicly available                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why composition rules backfire:**

```
┌─────────────────────────────────────────────────────────────────┐
│          COMPOSITION RULES: GOOD INTENTIONS, BAD OUTCOMES       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Rule: "Must contain uppercase, lowercase, digit, symbol"       │
│                                                                 │
│  What you HOPE users create:                                    │
│  └─ "k7#mP2$xR9nQ"  (strong, random)                           │
│                                                                 │
│  What users ACTUALLY create:                                    │
│  ├─ "Password1!"                                                │
│  ├─ "Welcome123#"                                               │
│  ├─ "Summer2024!"                                               │
│  └─ "Qwerty1@"                                                  │
│                                                                 │
│  These all "pass" the rules but are in every attacker's         │
│  wordlist. The rules create a FALSE sense of security           │
│  and annoy users into choosing predictable patterns.            │
│                                                                 │
│  A passphrase like "correct horse battery staple" (30+ chars)   │
│  is vastly stronger than "P@ssw0rd!" (9 chars with rules).     │
│  But composition rules don't allow the passphrase without       │
│  numbers and symbols.                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical implementation — breached password check:**

```python
# app/schemas/user.py (updated)

# A small sample for demonstration. In production, use the
# Have I Been Pwned API (https://haveibeenpwned.com/API/v3)
# with k-anonymity to check without sending the full password.
COMMON_PASSWORDS = {
    "password", "123456", "12345678", "qwerty", "abc123",
    "monkey", "1234567", "letmein", "trustno1", "dragon",
    "baseball", "iloveyou", "master", "sunshine", "ashley",
    "football", "shadow", "password1", "password123",
    "welcome", "welcome1",
}


class UserCreate(BaseModel):
    email: EmailStr
    username: str = Field(
        min_length=3, max_length=50, pattern=r"^[a-zA-Z0-9_]+$"
    )
    password: str = Field(min_length=8, max_length=128)

    @field_validator("password")
    @classmethod
    def password_not_common(cls, v: str) -> str:
        if v.lower() in COMMON_PASSWORDS:
            raise ValueError(
                "This password is too common and has appeared in data breaches"
            )
        return v

    @field_validator("password")
    @classmethod
    def password_not_username(cls, v: str, info) -> str:
        if "username" in info.data and info.data["username"].lower() in v.lower():
            raise ValueError("Password must not contain your username")
        return v
```

---

# PART 5: COMMON MISTAKES AND SECURITY MINDSET

## 5.1 Error Messages That Leak Information

**This is the most common authentication mistake beginners make.**

```python
# ❌ WRONG: Different errors for different cases
@router.post("/login")
async def bad_login(credentials: LoginRequest, session: AsyncSession = Depends(get_session)):
    result = await session.execute(
        select(User).where(User.username == credentials.username)
    )
    user = result.scalar_one_or_none()

    if not user:
        raise HTTPException(status_code=404, detail="User not found")  # ❌ LEAKED!

    if not verify_password(credentials.password, user.password_hash):
        raise HTTPException(status_code=401, detail="Wrong password")  # ❌ LEAKED!

    return {"message": "OK"}


# The attacker now has a USER ENUMERATION oracle:
# POST /login {"username": "alice", "password": "x"}
#   → 401 "Wrong password"    ← Alice EXISTS!
# POST /login {"username": "admin", "password": "x"}
#   → 404 "User not found"   ← No admin account.
# POST /login {"username": "bob", "password": "x"}
#   → 401 "Wrong password"    ← Bob EXISTS! Now brute-force bob.
```

```python
# ✅ CORRECT: Same error regardless of which check fails
@router.post("/login")
async def good_login(credentials: LoginRequest, session: AsyncSession = Depends(get_session)):
    result = await session.execute(
        select(User).where(User.username == credentials.username)
    )
    user = result.scalar_one_or_none()

    if not user or not verify_password(credentials.password, user.password_hash):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",  # ✅ Same message. Always.
        )

    return {"message": "OK"}
```

**The same principle applies to registration:**

```python
# ❌ WRONG: Tells attacker which emails are registered
raise HTTPException(detail="Email already registered")

# ✅ BETTER: On sensitive applications, return success either way
# and send a confirmation email (only the real owner sees it).
# This prevents the registration form from being an email oracle.
# For our course project, 409 Conflict is acceptable.
```

---

## 5.2 Never Log Passwords

```python
import logging
logger = logging.getLogger(__name__)

# ❌ NEVER DO THIS:
@router.post("/login")
async def terrible_login(credentials: LoginRequest, ...):
    logger.info(f"Login attempt: {credentials}")
    # Logs: Login attempt: username='alice' password='dolphin42'
    # Now the password is in your log files.
    # Log files get shipped to ELK, Datadog, CloudWatch...
    # Everyone with log access can read passwords.

# ❌ ALSO BAD:
@router.post("/login")
async def still_bad_login(credentials: LoginRequest, ...):
    logger.debug(f"Login with password: {credentials.password}")
    # "It's debug level, it won't show in production!"
    # Famous last words.

# ✅ CORRECT: Log the action, not the credentials
@router.post("/login")
async def good_login(credentials: LoginRequest, ...):
    logger.info(f"Login attempt for username: {credentials.username}")
    # Log WHAT happened, WHO attempted it, never the password.
```

```
┌─────────────────────────────────────────────────────────────────┐
│              LOGGING RULES FOR AUTH                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ LOG:                                                        │
│  ├─ "Login attempt for username: alice"                         │
│  ├─ "Registration attempt for email: a***@example.com"          │
│  ├─ "Failed login for username: alice (attempt #3)"             │
│  └─ "Password change for user_id: 550e8400-..."                 │
│                                                                 │
│  ❌ NEVER LOG:                                                  │
│  ├─ The password itself (plaintext)                             │
│  ├─ The password hash (helps offline attacks)                   │
│  ├─ The full email address (privacy, use masking)               │
│  └─ The request body raw (may contain password)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Timing Attacks (Brief)

> "This is a subtle attack. It exploits the fact that string comparison can fail at different speeds."

```python
# Standard string comparison short-circuits:
"dolphin42" == "xxxxxxx42"
#  d != x → returns False immediately (fast)

"dolphin42" == "dolphin43"
#  d==d, o==o, l==l, p==p, h==h, i==i, n==n, 4==4, 2!=3 → returns False (slower)
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   TIMING ATTACK                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  If the attacker can measure response time precisely:           │
│                                                                 │
│  "xxxxxxxx" → 0.100ms (fails on first char)                    │
│  "dxxxxxxx" → 0.101ms (matches 'd', fails on second)           │
│  "doxxxxxx" → 0.102ms (matches 'do', fails on third)           │
│                                                                 │
│  Microsecond by microsecond, they reconstruct the value.        │
│                                                                 │
│                                                                 │
│  WHY WE DON'T WORRY MUCH:                                      │
│  ─────────────────────────                                      │
│  pwdlib's verify() uses constant-time comparison internally.    │
│  Argon2id's deliberate slowness (~200ms) dwarfs any timing      │
│  differences from string comparison.                            │
│                                                                 │
│  But you should know the concept exists:                        │
│  ├─ Never compare hashes with == directly                       │
│  ├─ Use hmac.compare_digest() for raw hash comparison           │
│  └─ Or just use pwdlib.verify() which handles this for you      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 The Security Mindset

> "Here is the most important thing from today's lecture. It's not code. It's a way of thinking."

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE SECURITY MINDSET                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ASSUME BREACH                                               │
│     Your database WILL be stolen.                               │
│     Design so that a stolen database is useless.                │
│                                                                 │
│  2. NEVER INVENT CRYPTO                                         │
│     Don't build your own hashing scheme.                        │
│     Don't "improve" Argon2id with extra steps.                  │
│     Don't hash the hash "for extra security."                   │
│     Use the library. Trust the mathematicians.                  │
│                                                                 │
│  3. MINIMIZE EXPOSURE                                           │
│     The password should exist in plaintext for the shortest     │
│     possible time. Hash it immediately. Never log it.           │
│     Never return it. Never store it.                            │
│                                                                 │
│  4. FAIL CLOSED                                                 │
│     If you're not SURE the user is authenticated,               │
│     they're NOT authenticated. Default to denial.               │
│                                                                 │
│  5. THINK LIKE AN ATTACKER                                      │
│     Every error message is information.                         │
│     Every timing difference is a signal.                        │
│     Every log entry is a potential leak.                        │
│     Ask: "If I were trying to break this, what would I try?"    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│               AUTHENTICATION QUICK REFERENCE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INSTALL:                                                       │
│      uv add "pwdlib[argon2]"                                    │
│                                                                 │
│  SETUP:                                                         │
│      from pwdlib import PasswordHash                            │
│      hasher = PasswordHash.recommended()   # Argon2id           │
│                                                                 │
│  HASH A PASSWORD (registration):                                │
│      hashed = hasher.hash("plaintext_password")                 │
│                                                                 │
│  VERIFY A PASSWORD (login):                                     │
│      is_valid = hasher.verify("plaintext", hashed)              │
│                                                                 │
│  VERIFY AND UPGRADE (algorithm migration):                      │
│      valid, new_hash = hasher.verify_and_update("plain", hashed)│
│                                                                 │
│                                                                 │
│  GOLDEN RULES:                                                  │
│      ✅ Column name: password_hash (not password)               │
│      ✅ Same error: "Invalid credentials" for all failures      │
│      ✅ Never log passwords or hashes                           │
│      ✅ Never return password_hash in API responses              │
│      ✅ Check against breached password lists                   │
│      ❌ Never store plaintext or encrypted passwords            │
│      ❌ Never use MD5/SHA for password hashing                  │
│      ❌ Never invent your own hashing scheme                    │
│      ❌ Never require periodic forced password changes          │
│                                                                 │
│                                                                 │
│  ALGORITHM HIERARCHY:                                           │
│      BEST:   Argon2id (memory-hard, PHC winner)                 │
│      GOOD:   Bcrypt (time-tested, widely deployed)              │
│      BAD:    SHA-256/SHA-512 (too fast, not designed for pwd)   │
│      AWFUL:  MD5 (too fast, collision-vulnerable)               │
│      DEATH:  Plaintext / reversible encryption                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  AUTHENTICATION = PROVING IDENTITY                              │
│                                                                 │
│  The server never stores your password.                         │
│  It stores a one-way, salted, deliberately slow hash.           │
│  At login, it re-hashes your input and compares.                │
│                                                                 │
│                                                                 │
│     Registration:                                               │
│     "dolphin42" ──▶ Argon2id(salt + "dolphin42")                │
│                        │                                        │
│                        ▼                                        │
│                 "$argon2id$v=19$..."  → stored in DB             │
│                                                                 │
│     Login:                                                      │
│     "dolphin42" ──▶ Argon2id(same salt + "dolphin42")           │
│                        │                                        │
│                        ▼                                        │
│                 "$argon2id$v=19$..."  → compare with DB          │
│                        │                                        │
│                    MATCH? ──▶ Authenticated ✅                   │
│                                                                 │
│                                                                 │
│  THE BUILDING SECURITY ANALOGY:                                 │
│  ├─ Security guard = Authentication middleware                  │
│  ├─ Your ID = username + password                               │
│  ├─ Guard's records = password_hash in PostgreSQL               │
│  ├─ Fingerprint scanner = Argon2id (verify without storing)     │
│  └─ "Invalid credentials" = "I won't tell you WHAT was wrong"  │
│                                                                 │
│                                                                 │
│  THE ARMS RACE WE WALKED THROUGH:                               │
│  ├─ Plaintext       → instant breach, NEVER use                 │
│  ├─ Encryption      → key gets stolen, same result              │
│  ├─ Fast hash (MD5) → billions of guesses/sec                   │
│  ├─ + Salt          → kills rainbow tables, not brute force     │
│  ├─ + Slow (bcrypt) → kills CPU brute force, not GPU            │
│  └─ + Memory-hard (Argon2id) → kills GPU attacks too ✅         │
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
│  WEEK 9 (Next lecture):                                         │
│  └─ LECTURE 2: JWT Authentication                               │
│     Right now, our login returns {"message": "Login successful"}│
│     That's useless — the client can't PROVE they logged in      │
│     on the next request. JWT tokens solve this.                 │
│     "Here's a signed token. Show it on every request."          │
│                                                                 │
│  WEEK 9, LECTURE 3:                                             │
│  └─ Protected Endpoints & RBAC                                  │
│     Use authentication to build authorization.                  │
│     get_current_user dependency.                                │
│     Role-based access control (admin, member, viewer).          │
│                                                                 │
│  WEEK 9, LECTURE 4:                                             │
│  └─ API Security                                                │
│     CORS, SQL injection, rate limiting on /auth/login,          │
│     OWASP Top 10. The security perimeter around auth.           │
│                                                                 │
│  WEEK 10 (Redis):                                               │
│  └─ Store refresh tokens in Redis with TTL.                     │
│     Fast lookup, automatic expiration, revocation on logout.    │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ Multi-tenant auth: users belong to organizations.           │
│     Row-level data isolation. Audit logging.                    │
│     Everything from today scales to production SaaS.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
