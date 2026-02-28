# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "Relationships are free Python attributes. Once I define `relationship()`, accessing `.posts` or `.author` is just like reading `.name` — instant, costless, and always available. The ORM handles everything automatically, so I just write normal Python and it works."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | `relationship()` creates a column in the database | `relationship()` is Python-only; only `ForeignKey` creates a real database column |
| 2 | Accessing `.posts` or `.author` is free (no query) | Each lazy access fires a SQL query; in a loop this becomes N+1 queries |
| 3 | `joinedload` returns one instance per parent object | The SQL JOIN duplicates parent rows; without `.unique()` you get duplicate references |
| 4 | `.join()` in a query includes all parents | `.join()` is an INNER JOIN; parents with no matching children are silently excluded |
| 5 | Eager loading applies globally once specified | Loading strategy is per-query via `.options()`; the same model can be loaded differently |
| 6 | Offset pagination gives stable results across pages | Inserts between page requests shift all offsets, causing duplicates or skipped rows |
| 7 | `joinedload` and `selectinload` are interchangeable | `joinedload` duplicates data and causes cartesian products; `selectinload` is safer for large sets |
| 8 | N+1 only matters with "big data" | Even 50 objects = 51 queries; at 5ms each that is 255ms — noticeable on every request |

**The Prediction Gap:** When shown code that iterates over authors and prints `author.posts`, students predict one query because in Python, attribute access is free. The actual behavior is 1 + N queries because every `.posts` access fires a separate `SELECT` to the database, silently destroying API response times.

---

## The Main Purpose (Essence)

**Core Insight:** Every relationship navigation has a database cost. The ORM lets you write Pythonic code (`author.posts`), but **you** must tell it **when** and **how** to load that data. The difference between a 3-query endpoint and a 101-query endpoint is a single `.options()` call.

**The Problems Being Solved:**

| Concept | Without Understanding | With Understanding |
|---------|----------------------|-------------------|
| `relationship()` | "I defined it, so the data is there" | "It's a bridge I must consciously choose when to cross" |
| Eager loading | "The ORM optimizes automatically" | "I choose joinedload (1 JOIN) or selectinload (2 queries via IN)" |
| `.join()` vs `.outerjoin()` | "Join gets me all the parents" | "INNER JOIN excludes parents with no children" |
| Keyset pagination | "OFFSET works, why change?" | "OFFSET scans from row 0 every time; keyset jumps via index" |

**The Key Distinction:**
- **ForeignKey** = a wire in the database (real column, enforces integrity)
- **relationship()** = a Python handle on that wire (not a column, fires queries)
- **Eager loading** = telling SQLAlchemy to pull the wire NOW, not later

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "joinedload deduplicates results" | `len(departments)` == 3 | `len(departments)` == 5 (parent repeated per child row) | SQL JOIN produces one row per child; `.scalars().all()` returns each row |
| "selectinload optimized the query" | Serializing 5 posts = 2 queries | 7 queries (2 for posts+tags, 5 for each author) | Eager loading is per-relationship; author was never eagerly loaded |
| ".join() is inclusive" | Query returns all 4 authors | Returns 3 authors (Diana excluded) | `.join()` is INNER JOIN; no children = no match = excluded |
| "Offset pagination is stable" | Page 1 and Page 2 have no overlap | 1 post appears in both pages | New insert shifts all offsets; boundary post slides into next page |

---

# SHARED SETUP

**All Level 1 exercises use the following database models and seed data unless stated otherwise. Run this first:**

```python
from sqlalchemy import (
    create_engine, select, ForeignKey, String, Text,
    Table, Column, func, event, and_, or_
)
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, relationship,
    Session, joinedload, selectinload
)
from datetime import datetime, timedelta
from typing import Optional


class Base(DeclarativeBase):
    pass


post_tags = Table(
    "post_tags",
    Base.metadata,
    Column("post_id", ForeignKey("posts.id"), primary_key=True),
    Column("tag_id", ForeignKey("tags.id"), primary_key=True),
)


class Author(Base):
    __tablename__ = "authors"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(200), unique=True)
    posts: Mapped[list["Post"]] = relationship(back_populates="author")
    def __repr__(self) -> str:
        return f"Author(id={self.id}, name='{self.name}')"


class Post(Base):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text)
    published_at: Mapped[datetime] = mapped_column()
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    author: Mapped["Author"] = relationship(back_populates="posts")
    tags: Mapped[list["Tag"]] = relationship(
        secondary=post_tags, back_populates="posts"
    )
    def __repr__(self) -> str:
        return f"Post(id={self.id}, title='{self.title}')"


class Tag(Base):
    __tablename__ = "tags"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)
    posts: Mapped[list["Post"]] = relationship(
        secondary=post_tags, back_populates="tags"
    )
    def __repr__(self) -> str:
        return f"Tag(id={self.id}, name='{self.name}')"


class QueryCounter:
    """Utility to count SQL queries for instrumentation exercises."""
    def __init__(self, engine):
        self.count = 0
        self.queries = []
        event.listen(engine, "before_cursor_execute", self._callback)

    def _callback(self, conn, cursor, statement, parameters, context, executemany):
        # Ignore SQLite internal commands
        if not statement.startswith("PRAGMA"):
            self.count += 1
            self.queries.append(statement)

    def reset(self):
        self.count = 0
        self.queries = []


def create_test_database():
    """Create and seed a test database. Returns the engine."""
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)

    with Session(engine) as session:
        # 4 authors — Diana has NO posts
        alice = Author(name="Alice", email="alice@blog.com")
        bob = Author(name="Bob", email="bob@blog.com")
        charlie = Author(name="Charlie", email="charlie@blog.com")
        diana = Author(name="Diana", email="diana@blog.com")

        # 4 tags
        python_tag = Tag(name="python")
        db_tag = Tag(name="database")
        tutorial_tag = Tag(name="tutorial")
        async_tag = Tag(name="async")

        # 6 posts with varied authors and dates
        base = datetime(2025, 1, 1)
        posts = [
            Post(title="Type Hints 101", content="...", author=alice,
                 published_at=base + timedelta(days=1),
                 tags=[python_tag, tutorial_tag]),
            Post(title="Async Deep Dive", content="...", author=alice,
                 published_at=base + timedelta(days=2),
                 tags=[python_tag, async_tag]),
            Post(title="SQLAlchemy Guide", content="...", author=alice,
                 published_at=base + timedelta(days=3),
                 tags=[python_tag, db_tag]),
            Post(title="REST API Design", content="...", author=bob,
                 published_at=base + timedelta(days=4),
                 tags=[tutorial_tag]),
            Post(title="Testing Strategies", content="...", author=bob,
                 published_at=base + timedelta(days=5),
                 tags=[python_tag, tutorial_tag]),
            Post(title="Database Indexing", content="...", author=charlie,
                 published_at=base + timedelta(days=6),
                 tags=[db_tag]),
        ]

        session.add_all(posts)
        session.add(diana)
        session.commit()

    return engine


# Create the engine for exercises
engine = create_test_database()
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEED DATA SUMMARY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AUTHORS:                                                       │
│    Alice   (id=1) → 3 posts                                    │
│    Bob     (id=2) → 2 posts                                    │
│    Charlie (id=3) → 1 post                                     │
│    Diana   (id=4) → 0 posts  ← IMPORTANT: no posts at all     │
│                                                                 │
│  POSTS (ordered by published_at):                               │
│    id=1  "Type Hints 101"      Alice    [python, tutorial]      │
│    id=2  "Async Deep Dive"     Alice    [python, async]         │
│    id=3  "SQLAlchemy Guide"    Alice    [python, database]      │
│    id=4  "REST API Design"     Bob      [tutorial]              │
│    id=5  "Testing Strategies"  Bob      [python, tutorial]      │
│    id=6  "Database Indexing"   Charlie  [database]              │
│                                                                 │
│  TAGS:                                                          │
│    python (4 posts), tutorial (3 posts),                        │
│    database (2 posts), async (1 post)                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Count That Lied

A developer wants to print all authors and how many posts each has written. They know `joinedload` is the efficient way to load related data, so they use it:

```python
with Session(engine) as session:
    stmt = select(Author).options(joinedload(Author.posts))
    authors = session.execute(stmt).scalars().all()

    print(f"Number of authors: {len(authors)}")
    print()
    for author in authors:
        print(f"  {author.name}: {len(author.posts)} posts")
```

**Questions:**

1. **Predict the output.** How many authors does `len(authors)` return? What does the loop print?

2. **Explain why.** The database has 4 authors. If the result list is not 4, what caused the discrepancy? Think about what SQL query `joinedload` generates and how many rows that query returns.

3. **Mental Trace:** Write out the SQL that `joinedload` produces (a LEFT OUTER JOIN). How many rows does that JOIN result in, given the seed data? Now trace how `.scalars().all()` converts those rows into a Python list. How many list elements do you get?

4. **Fix the code.** What single method call makes this code produce the correct count?

---

## Exercise 1.2: The Partial Optimization

A developer learned about N+1 queries and added eager loading. But they only tested whether the tags loaded correctly. A code reviewer sees no performance issues in the test suite, yet the endpoint is slow:

```python
def post_to_api_response(post: Post) -> dict:
    """Serialize a Post for the JSON API response."""
    return {
        "id": post.id,
        "title": post.title,
        "published_at": post.published_at.isoformat(),
        "author_name": post.author.name,          # Line A
        "tags": [tag.name for tag in post.tags],   # Line B
    }


with Session(engine) as session:
    # "I added selectinload so it's optimized now!"
    stmt = (
        select(Post)
        .options(selectinload(Post.tags))
        .order_by(Post.published_at.desc())
    )
    posts = session.execute(stmt).scalars().all()

    # Serialize for API response
    response = [post_to_api_response(p) for p in posts]

    print(f"Serialized {len(response)} posts")
    for item in response:
        print(f"  '{item['title']}' by {item['author_name']} | tags: {item['tags']}")
```

**Questions:**

1. **Predict the total number of SQL queries** this code executes. The developer expects 2 (one for posts, one for tags via `selectinload`). Are they correct?

2. **Identify the hidden cost.** Which line inside `post_to_api_response` triggers additional SQL queries? Why doesn't `selectinload(Post.tags)` protect it?

3. **Explain the principle.** The developer assumed that using `.options(selectinload(...))` makes the entire query "optimized." What is wrong with this mental model? Where does the loading strategy actually live — on the model, on the session, or on the individual query?

4. **Calculate the exact query count** given the seed data (6 posts, 3 unique authors: Alice with 3 posts, Bob with 2, Charlie with 1). Be careful — SQLAlchemy's identity map caches objects within a session. Does the second access to Alice's author fire a query?

---

## Exercise 1.3: The Vanishing Author

A developer is building an author directory page. They want to show each author alongside the title of one of their posts. They use `.join()` to access post data in the query:

```python
with Session(engine) as session:
    # "Join authors with their posts so I can filter/access post data"
    stmt = (
        select(Author)
        .join(Author.posts)
    )
    authors = session.execute(stmt).scalars().unique().all()

    print(f"Total authors in database: 4")
    print(f"Authors returned by query: {len(authors)}")
    print()
    for author in authors:
        print(f"  {author.name}")
```

**Questions:**

1. **Predict the output.** How many authors does the query return? Which author names are printed?

2. **Identify the missing author.** One of the four seeded authors does not appear. Who, and why?

3. **Explain the SQL.** What type of JOIN does `.join(Author.posts)` produce? Write out the SQL and trace which rows the database returns. What happens to an author who has zero matching rows in the `posts` table?

4. **The fix.** How would you modify the query to include ALL authors, even those with no posts? (Hint: Think about the difference between `JOIN` and `LEFT JOIN` in your Week 5 SQL knowledge.)

---

## Exercise 1.4: The Sliding Window

A developer implements offset-based pagination for a blog feed. During testing, a user reports seeing the same post on two different pages:

```python
def create_pagination_database():
    """Create a database with 20 posts for pagination testing."""
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)

    with Session(engine) as session:
        author = Author(name="Alice", email="alice@blog.com")
        session.add(author)
        session.flush()

        for i in range(1, 21):
            post = Post(
                title=f"Post {i}",
                content=f"Content for post {i}",
                author_id=author.id,
                published_at=datetime(2025, 1, i),  # Post 20 is newest
            )
            session.add(post)
        session.commit()

    return engine


pagination_engine = create_pagination_database()

# ── User opens the blog, requests page 1 ──
with Session(pagination_engine) as session:
    stmt = (
        select(Post)
        .order_by(Post.published_at.desc())
        .limit(5)
        .offset(0)
    )
    page1 = session.execute(stmt).scalars().all()
    page1_titles = [p.title for p in page1]

print(f"Page 1: {page1_titles}")

# ── While user reads page 1, another author publishes a new post ──
with Session(pagination_engine) as session:
    author = session.execute(select(Author)).scalars().first()
    new_post = Post(
        title="BREAKING NEWS",
        content="Just published!",
        author_id=author.id,
        published_at=datetime(2025, 1, 25),  # Newest post in the database
    )
    session.add(new_post)
    session.commit()

# ── User clicks "Next Page" ──
with Session(pagination_engine) as session:
    stmt = (
        select(Post)
        .order_by(Post.published_at.desc())
        .limit(5)
        .offset(5)
    )
    page2 = session.execute(stmt).scalars().all()
    page2_titles = [p.title for p in page2]

print(f"Page 2: {page2_titles}")

# Check for duplicates
overlap = set(page1_titles) & set(page2_titles)
print(f"Posts appearing on BOTH pages: {overlap}")
```

**Questions:**

1. **Predict the output.** What are the 5 titles on page 1? What are the 5 titles on page 2? Is there any overlap?

2. **Trace the OFFSET shift.** Before the insert, the database order (newest first) is `[Post 20, Post 19, ..., Post 1]`. After inserting "BREAKING NEWS," what is the new order? When the user requests `OFFSET 5`, which posts occupy positions 5-9?

3. **Explain the fundamental flaw.** Why does OFFSET-based pagination break when data changes between page requests? Is this a bug in SQLAlchemy, or a property of how `OFFSET` works at the SQL level?

4. **Propose a fix.** How would keyset (cursor) pagination avoid this problem? What would the "page 2" query look like if instead of `OFFSET 5`, you used the last post's ID from page 1 as a cursor?

---

# Level 1 Solutions

## Solution 1.1: The Count That Lied

**1. Output:**
```
Number of authors: 7
  
  Alice: 3 posts
  Alice: 3 posts
  Alice: 3 posts
  Bob: 2 posts
  Bob: 2 posts
  Charlie: 1 posts
  Diana: 0 posts
```

**2. Explanation:**

`joinedload` generates a `LEFT OUTER JOIN`:

```sql
SELECT authors.id, authors.name, authors.email,
       posts.id, posts.title, posts.content, posts.published_at, posts.author_id
FROM authors
LEFT OUTER JOIN posts ON authors.id = posts.author_id
```

This JOIN produces **one row per post**, not one row per author:

| author.id | author.name | post.id | post.title |
|-----------|-------------|---------|------------|
| 1 | Alice | 1 | Type Hints 101 |
| 1 | Alice | 2 | Async Deep Dive |
| 1 | Alice | 3 | SQLAlchemy Guide |
| 2 | Bob | 4 | REST API Design |
| 2 | Bob | 5 | Testing Strategies |
| 3 | Charlie | 6 | Database Indexing |
| 4 | Diana | NULL | NULL |

**7 rows.** `.scalars().all()` converts each row into a Python object. SQLAlchemy's identity map ensures that all three "Alice" rows reference the **same** `Author` Python object, but the **list** still contains 7 entries — three of which are the same Alice object.

The `joinedload` correctly populates each author's `.posts` collection (Alice gets all 3 posts attached). The problem is not with the *relationship loading* — it's with the *result set containing duplicates*.

**3. Mental Trace:**
```
Row 1 → Author(id=1, "Alice")   → new object, starts building posts list
Row 2 → Author(id=1, "Alice")   → same object (identity map), adds post
Row 3 → Author(id=1, "Alice")   → same object, adds post
Row 4 → Author(id=2, "Bob")     → new object
Row 5 → Author(id=2, "Bob")     → same object
Row 6 → Author(id=3, "Charlie") → new object
Row 7 → Author(id=4, "Diana")   → new object, LEFT JOIN → NULL post → empty list

.scalars().all() returns: [Alice, Alice, Alice, Bob, Bob, Charlie, Diana]
len() = 7, not 4!
```

**4. The Fix:**
```python
authors = session.execute(stmt).scalars().unique().all()
#                                        ^^^^^^^^
# .unique() deduplicates by identity → [Alice, Bob, Charlie, Diana]
# len() = 4 ✅
```

**The Key Insight:** `joinedload` trades duplication for efficiency — it loads everything in one query, but the JOIN multiplies rows. `.unique()` is the mandatory cleanup step. Forgetting it doesn't crash; it silently corrupts your counts and iterations.

---

## Solution 1.2: The Partial Optimization

**1. Total queries: not 2, but 5.**

The developer is wrong. The breakdown:
- Query 1: `SELECT * FROM posts ORDER BY published_at DESC` (the main select)
- Query 2: `SELECT * FROM tags JOIN post_tags WHERE post_id IN (1,2,3,4,5,6)` (selectinload for tags)
- Query 3: `SELECT * FROM authors WHERE id = 3` (lazy load Charlie's author — post 6)
- Query 4: `SELECT * FROM authors WHERE id = 2` (lazy load Bob's author — post 5)
- Query 5: `SELECT * FROM authors WHERE id = 1` (lazy load Alice's author — post 4... wait, actually the order matters)

Let me trace more carefully with `published_at DESC`:

Posts in order: `[Post 6 (Charlie), Post 5 (Bob), Post 4 (Bob), Post 3 (Alice), Post 2 (Alice), Post 1 (Alice)]`

Serialization processes them in this order:
- Post 6: `post.author.name` → Charlie not in identity map → **Query 3**: load Charlie
- Post 5: `post.author.name` → Bob not in identity map → **Query 4**: load Bob
- Post 4: `post.author.name` → Bob IS in identity map → **0 queries** (cached!)
- Post 3: `post.author.name` → Alice not in identity map → **Query 5**: load Alice
- Post 2: `post.author.name` → Alice IS in identity map → 0 queries
- Post 1: `post.author.name` → Alice IS in identity map → 0 queries

**Total: 2 (posts + tags) + 3 (unique authors) = 5 queries.**

**2. The hidden cost:**

**Line A** (`post.author.name`) triggers the extra queries. `selectinload(Post.tags)` only tells SQLAlchemy to eagerly load the `tags` relationship. It says nothing about the `author` relationship. Each relationship must be independently configured for eager loading.

**3. The principle:**

Loading strategies are **per-relationship, per-query**. They are not global, not on the model, and not on the session. Writing `.options(selectinload(Post.tags))` means "when executing **this specific statement**, eagerly load `.tags`." The `.author` relationship is untouched — it remains on its default lazy loading strategy, which fires a query on first access.

**4. Exact query count:**

5 queries, not 2. The identity map saves us from the worst case (which would be 2 + 6 = 8 if every `.author` access fired a query). Since SQLAlchemy caches loaded objects within a session, only the **first** access to each unique Author fires a query. Three unique authors = 3 lazy queries.

With 1000 posts by 200 unique authors, this would be 2 + 200 = 202 queries, not 1002.

**The Fix:**
```python
stmt = (
    select(Post)
    .options(
        selectinload(Post.tags),
        selectinload(Post.author),   # ← Add this!
    )
    .order_by(Post.published_at.desc())
)
# Now: 3 queries total (posts, tags via IN, authors via IN). Always 3, regardless of row count.
```

**The Key Insight:** Eager loading is a **per-relationship opt-in**, not a global switch. Seeing `.options(selectinload(...))` in code does NOT mean "this query is optimized." You must check that **every relationship accessed downstream** has its own loading strategy.

---

## Solution 1.3: The Vanishing Author

**1. Output:**
```
Total authors in database: 4
Authors returned by query: 3

  Alice
  Bob
  Charlie
```

**Diana is missing.**

**2. Why Diana vanishes:**

Diana has zero posts. The query uses `.join(Author.posts)`, which generates an **INNER JOIN**:

```sql
SELECT authors.id, authors.name, authors.email
FROM authors
JOIN posts ON authors.id = posts.author_id
```

An INNER JOIN requires a match on **both** sides. Diana has no rows in `posts` where `author_id = 4`, so she produces zero matching rows and is excluded from the result entirely.

**3. SQL trace:**

The JOIN produces these rows:

| author.id | author.name | post.id |
|-----------|-------------|---------|
| 1 | Alice | 1 |
| 1 | Alice | 2 |
| 1 | Alice | 3 |
| 2 | Bob | 4 |
| 2 | Bob | 5 |
| 3 | Charlie | 6 |

Six rows. Diana does not appear because there is no `posts` row with `author_id = 4`. `.unique()` deduplicates to 3 authors: Alice, Bob, Charlie.

**4. The Fix:**
```python
stmt = (
    select(Author)
    .outerjoin(Author.posts)   # ← LEFT OUTER JOIN instead of INNER JOIN
)
authors = session.execute(stmt).scalars().unique().all()
# Now returns 4 authors, including Diana
```

The `LEFT OUTER JOIN` keeps all rows from the left table (authors), even if there's no match in the right table (posts). Diana appears with NULL post columns — but she appears.

**The Key Insight:** `.join()` is INNER JOIN. It is a filter disguised as a join — it silently removes parents with no children. This is correct when you *want* to filter (e.g., "find authors who have written about databases"), but incorrect when you want all parents (e.g., "show all authors"). Always ask: *"Should empty parents be included?"* If yes, use `.outerjoin()`.

---

## Solution 1.4: The Sliding Window

**1. Output:**
```
Page 1: ['Post 20', 'Post 19', 'Post 18', 'Post 17', 'Post 16']
Page 2: ['Post 16', 'Post 15', 'Post 14', 'Post 13', 'Post 12']
Posts appearing on BOTH pages: {'Post 16'}
```

**Post 16 appears on both pages.**

**2. The OFFSET shift:**

Before the insert, the order (newest first) is:
```
Position 0: Post 20
Position 1: Post 19
Position 2: Post 18
Position 3: Post 17
Position 4: Post 16   ← Page 1 boundary
Position 5: Post 15
...
```

Page 1 (`OFFSET 0, LIMIT 5`) returns: `[Post 20, 19, 18, 17, 16]`

After inserting "BREAKING NEWS" (dated Jan 25, newer than all):
```
Position 0: BREAKING NEWS   ← NEW! Everything below shifts down by 1
Position 1: Post 20
Position 2: Post 19
Position 3: Post 18
Position 4: Post 17
Position 5: Post 16         ← Was position 4, now position 5
Position 6: Post 15
...
```

Page 2 (`OFFSET 5, LIMIT 5`) now returns: `[Post 16, 15, 14, 13, 12]`

Post 16 was at position 4 during page 1, but shifted to position 5 after the insert — landing it right at the start of page 2.

**3. The fundamental flaw:**

This is not a SQLAlchemy bug — it is a property of how `OFFSET` works at the SQL level. `OFFSET N` means "skip the first N rows **as they exist right now**." If the data changes between requests, "the first N rows" changes too. OFFSET has no memory of what it previously returned.

**4. Keyset pagination fix:**

Instead of OFFSET, use the **last seen value** as a cursor:

```python
# Page 1: same as before
# Last post on page 1: Post 16 (published_at = Jan 16)

# Page 2 using keyset:
last_seen_date = datetime(2025, 1, 16)  # From page 1's last item
last_seen_id = 16

stmt = (
    select(Post)
    .where(
        or_(
            Post.published_at < last_seen_date,
            and_(
                Post.published_at == last_seen_date,
                Post.id < last_seen_id
            )
        )
    )
    .order_by(Post.published_at.desc(), Post.id.desc())
    .limit(5)
)
# Returns: [Post 15, 14, 13, 12, 11] — no matter what was inserted!
```

The keyset query says "give me posts **older** than the last one I saw." Inserting a **newer** post doesn't affect this — the cursor position is anchored to a value, not a row count.

**The Key Insight:** OFFSET counts from the beginning every time — it has no anchor. Keyset pagination uses an anchored position (a value) that doesn't shift when data changes. The tradeoff: you lose "jump to page 50" ability, but gain consistency and O(log n) performance at any depth.

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Dashboard That Crawled

**Context:** You're building a blog dashboard that shows all authors, their posts, and the tags on each post. The code works correctly — every author, post, and tag appears in the output. But the page takes 3 seconds to load in production with 50 authors, 500 posts, and 200 tags.

```python
def build_blog_dashboard(session: Session) -> list[dict]:
    """Build a dashboard: all authors → their posts → each post's tags."""
    stmt = select(Author).order_by(Author.name)
    authors = session.execute(stmt).scalars().all()

    dashboard = []
    for author in authors:
        author_data = {
            "name": author.name,
            "email": author.email,
            "posts": [],
        }

        for post in author.posts:                          # Line A
            post_data = {
                "title": post.title,
                "published_at": post.published_at.isoformat(),
                "tags": [tag.name for tag in post.tags],   # Line B
            }
            author_data["posts"].append(post_data)

        dashboard.append(author_data)

    return dashboard


# Test it
with Session(engine) as session:
    dashboard = build_blog_dashboard(session)
    for entry in dashboard:
        print(f"\n{entry['name']}:")
        for post in entry["posts"]:
            print(f"  '{post['title']}' — tags: {post['tags']}")
```

---

**Task 2a: Identify the Flaw**

The output is correct — every author, post, and tag appears. The function works. But:

1. How many SQL queries does this function execute against the seed database (4 authors, 6 posts, varied tags)?
2. How many queries would it execute with 50 authors, 500 posts (10 per author), and each post having 3 tags?
3. Which two lines are responsible?

Do NOT fix the code yet.

---

**Task 2b: Explain the Mechanism**

1. What loading strategy is `author.posts` using? (You did not specify one.)
2. What loading strategy is `post.tags` using?
3. Explain how N+1 compounds when it occurs at **two nesting levels**. Write the formula for total queries.

---

**Task 2c: Instrument to Prove Diagnosis**

Use the `QueryCounter` from the shared setup to measure the exact query count:

```python
counter = QueryCounter(engine)

with Session(engine) as session:
    dashboard = build_blog_dashboard(session)

print(f"\nTotal queries: {counter.count}")
print(f"Query breakdown:")
for i, q in enumerate(counter.queries, 1):
    # Print just the first 80 characters of each query
    print(f"  Query {i}: {q[:80]}...")
```

Run this and confirm your prediction from Task 2a. Categorize each query: is it the initial select, a post load, or a tag load?

---

**Task 2d: Fix the Code**

Rewrite `build_blog_dashboard` so it produces the **exact same output** but executes at most **3 SQL queries**, regardless of how many authors, posts, or tags exist.

Requirements:
- The function signature must not change
- The output dictionary structure must not change
- Use eager loading

---

**Task 2e: Write Verification Test**

Write an assertion that **fails** on the original code and **passes** on your fixed version:

```python
def test_dashboard_query_efficiency(engine):
    counter = QueryCounter(engine)
    counter.reset()

    with Session(engine) as session:
        dashboard = build_blog_dashboard(session)

    # Your assertion here: the original fails this, your fix passes it
    assert counter.count <= ???, f"Expected ≤ ??? queries, got {counter.count}"
    
    # Also verify correctness is preserved
    author_names = {entry["name"] for entry in dashboard}
    assert "Alice" in author_names
    assert "Diana" in author_names  # Diana has 0 posts but should still appear
```

---

## Exercise 2.2: The Report with Ghost Authors

**Context:** A reporting endpoint generates a summary of all authors and how many posts each has written. The marketing team notices that Diana — who just joined the platform and hasn't published yet — is missing from the report entirely. "She signed up last week! Why isn't she listed?"

```python
def generate_author_report(session: Session) -> list[dict]:
    """Generate author report with post counts."""
    stmt = (
        select(Author)
        .join(Author.posts)
        .order_by(Author.name)
    )
    authors = session.execute(stmt).scalars().unique().all()

    report = []
    for author in authors:
        report.append({
            "name": author.name,
            "post_count": len(author.posts),
        })

    return report


with Session(engine) as session:
    report = generate_author_report(session)
    print("Author Report:")
    for entry in report:
        print(f"  {entry['name']}: {entry['post_count']} posts")
```

---

**Task 2a: Identify the Flaw**

1. Run the code. Which authors appear in the report? Who is missing?
2. The code has no errors, no exceptions, and the counts for the listed authors are correct. What is the **specific line** that causes Diana to vanish?
3. Does changing `Author.posts` to `Post.author` in the join change the result?

---

**Task 2b: Explain the Mechanism**

1. What SQL does `.join(Author.posts)` generate? Write it out.
2. Explain, in terms of the SQL JOIN, why Diana produces zero rows.
3. A junior developer suggests "just add Diana manually after the query." Why is this a bad fix?

---

**Task 2c: Instrument to Prove Diagnosis**

Add logging that reveals the discrepancy:

```python
with Session(engine) as session:
    # Count ALL authors in the database
    total_authors = session.execute(
        select(func.count(Author.id))
    ).scalar()

    report = generate_author_report(session)

    print(f"Authors in database: {total_authors}")
    print(f"Authors in report: {len(report)}")
    print(f"Missing: {total_authors - len(report)} author(s)")
    
    # Find who's missing
    reported_names = {entry["name"] for entry in report}
    all_names_stmt = select(Author.name)
    all_names = set(session.execute(all_names_stmt).scalars().all())
    missing = all_names - reported_names
    print(f"Missing authors: {missing}")
```

---

**Task 2d: Fix the Code**

Fix `generate_author_report` so that:
- ALL authors appear, including those with zero posts
- Authors with zero posts show `post_count: 0`
- The report is still ordered by name
- No N+1 queries (use eager loading for the post count)

---

**Task 2e: Write Verification Test**

```python
def test_report_includes_all_authors(engine):
    with Session(engine) as session:
        report = generate_author_report(session)

    names = {entry["name"] for entry in report}

    # This FAILS on the original code (Diana missing)
    # and PASSES on the fixed code
    assert "Diana" in names, "Diana is missing from the report!"
    assert len(report) == 4, f"Expected 4 authors, got {len(report)}"

    # Verify Diana shows 0 posts
    diana_entry = next(e for e in report if e["name"] == "Diana")
    assert diana_entry["post_count"] == 0
```

---

# Level 2 Solutions

## Solution 2.1: The Dashboard That Crawled

**Task 2a: Identify the Flaw**

With the seed data (4 authors, 6 posts):
- Query 1: `SELECT * FROM authors ORDER BY name` (get all authors)
- Queries 2-5: `SELECT * FROM posts WHERE author_id = ?` for Alice, Bob, Charlie, Diana → **4 queries** (Line A)
  - Diana's query returns 0 rows but still fires
- Queries 6-11: `SELECT tags.* FROM tags JOIN post_tags WHERE post_id = ?` for each of the 6 posts → **6 queries** (Line B)
- **Total: 1 + 4 + 6 = 11 queries**

With 50 authors, 500 posts (10 each), 3 tags per post:
- 1 (authors) + 50 (posts per author) + 500 (tags per post) = **551 queries**
- At 5ms per query: **2.75 seconds** — that's the 3-second page load!

The formula: **1 + A + P** where A = number of authors, P = total number of posts.

**Task 2b: Explain the Mechanism**

1. `author.posts` uses **lazy loading** (the default). No `.options()` was specified.
2. `post.tags` also uses **lazy loading** for the same reason.
3. N+1 at two levels compounds **additively, not multiplicatively**:
   - Level 1: 1 query (authors) + N queries (posts for each author) = 1 + N
   - Level 2: For each of the P total posts, 1 query for tags = P
   - Total: 1 + N + P
   - This is still linear, but with large datasets, it's devastating.

**Task 2c: Instrumentation Output**

```
Total queries: 11
Query breakdown:
  Query 1:  SELECT authors.id, authors.name, authors.email FROM authors ORDER BY...
  Query 2:  SELECT posts.id, ... FROM posts WHERE posts.author_id = ?...     (Alice)
  Query 3:  SELECT posts.id, ... FROM posts WHERE posts.author_id = ?...     (Bob)
  Query 4:  SELECT posts.id, ... FROM posts WHERE posts.author_id = ?...     (Charlie)
  Query 5:  SELECT posts.id, ... FROM posts WHERE posts.author_id = ?...     (Diana)
  Query 6:  SELECT tags.id, ... FROM tags JOIN post_tags ... WHERE post_tags.post_id = ?...
  Query 7:  SELECT tags.id, ... FROM tags JOIN post_tags ... WHERE post_tags.post_id = ?...
  Query 8:  SELECT tags.id, ... FROM tags JOIN post_tags ... WHERE post_tags.post_id = ?...
  Query 9:  SELECT tags.id, ... FROM tags JOIN post_tags ... WHERE post_tags.post_id = ?...
  Query 10: SELECT tags.id, ... FROM tags JOIN post_tags ... WHERE post_tags.post_id = ?...
  Query 11: SELECT tags.id, ... FROM tags JOIN post_tags ... WHERE post_tags.post_id = ?...
```

Queries 2-5: Lazy loads for `author.posts` (one per author).
Queries 6-11: Lazy loads for `post.tags` (one per post).

**Task 2d: The Fix**

```python
def build_blog_dashboard(session: Session) -> list[dict]:
    """Build a dashboard — now with eager loading."""
    stmt = (
        select(Author)
        .options(
            selectinload(Author.posts).selectinload(Post.tags)
            #                         ^^^^^^^^^^^^^^^^^^^^^^^^
            #  Chained! "When loading posts, also load each post's tags."
        )
        .order_by(Author.name)
    )
    authors = session.execute(stmt).scalars().all()

    # The rest of the function is UNCHANGED.
    # The loops still look the same — but no lazy queries fire.
    dashboard = []
    for author in authors:
        author_data = {
            "name": author.name,
            "email": author.email,
            "posts": [],
        }
        for post in author.posts:                          # No query — already loaded
            post_data = {
                "title": post.title,
                "published_at": post.published_at.isoformat(),
                "tags": [tag.name for tag in post.tags],   # No query — already loaded
            }
            author_data["posts"].append(post_data)
        dashboard.append(author_data)

    return dashboard
```

This generates exactly **3 queries**:
1. `SELECT * FROM authors ORDER BY name`
2. `SELECT * FROM posts WHERE author_id IN (1, 2, 3, 4)`
3. `SELECT tags.*, post_tags.* FROM tags JOIN post_tags WHERE post_id IN (1, 2, 3, 4, 5, 6)`

**Always 3 queries**, whether you have 4 authors or 4,000.

**Task 2e: Verification Test**

```python
def test_dashboard_query_efficiency(engine):
    counter = QueryCounter(engine)
    counter.reset()

    with Session(engine) as session:
        dashboard = build_blog_dashboard(session)

    assert counter.count <= 3, f"Expected ≤ 3 queries, got {counter.count}"

    # Verify correctness
    author_names = {entry["name"] for entry in dashboard}
    assert author_names == {"Alice", "Bob", "Charlie", "Diana"}

    alice = next(e for e in dashboard if e["name"] == "Alice")
    assert len(alice["posts"]) == 3
    assert "python" in alice["posts"][0]["tags"]  # Tags still loaded correctly
```

---

## Solution 2.2: The Report with Ghost Authors

**Task 2a: Identify the Flaw**

Output:
```
Author Report:
  Alice: 3 posts
  Bob: 2 posts
  Charlie: 1 posts
```

Diana is missing. The flaw is on this line:

```python
.join(Author.posts)
```

This generates an **INNER JOIN**. Diana has no posts, so no rows in `posts` have `author_id = 4`. The INNER JOIN produces zero rows for Diana, and she disappears.

Changing to `Post.author` in the join wouldn't help — it's still an INNER JOIN, just expressed differently.

**Task 2b: Explain the Mechanism**

The SQL generated:
```sql
SELECT authors.id, authors.name, authors.email
FROM authors
JOIN posts ON authors.id = posts.author_id
ORDER BY authors.name
```

This produces:

| author.id | author.name | post.id |
|-----------|-------------|---------|
| 1 | Alice | 1 |
| 1 | Alice | 2 |
| 1 | Alice | 3 |
| 2 | Bob | 4 |
| 2 | Bob | 5 |
| 3 | Charlie | 6 |

Diana is not here. Zero rows match `posts.author_id = 4`, so the INNER JOIN discards her.

"Just add Diana manually" is a bad fix because:
- It hardcodes a specific author rather than solving the general problem
- Any **future** author with zero posts would also be missing
- It introduces a maintenance burden and a race condition

**Task 2d: The Fix**

Two things need fixing: the JOIN type AND the loading strategy (to avoid N+1 on `len(author.posts)`):

```python
def generate_author_report(session: Session) -> list[dict]:
    """Generate author report — includes ALL authors."""
    stmt = (
        select(Author)
        .outerjoin(Author.posts)          # ← LEFT OUTER JOIN: keeps Diana
        .options(selectinload(Author.posts))  # ← Eager load to avoid N+1 on len()
        .order_by(Author.name)
    )
    authors = session.execute(stmt).scalars().unique().all()

    report = []
    for author in authors:
        report.append({
            "name": author.name,
            "post_count": len(author.posts),  # 0 for Diana, no extra query
        })

    return report
```

Note: In this specific case, you could also remove the `.outerjoin()` entirely and just use `select(Author).options(selectinload(Author.posts))`. The `.outerjoin()` was only needed because the original code used `.join()` to filter. Without the join, all authors are returned by default. But knowing when `.outerjoin()` is needed is essential — for example, if you want to **filter** by post attributes while still including authors with no matching posts, you'd need the outerjoin.

An even more efficient approach using a database-level count (avoids loading all post objects just to count them):

```python
from sqlalchemy import func, literal_column

def generate_author_report(session: Session) -> list[dict]:
    stmt = (
        select(
            Author.name,
            func.count(Post.id).label("post_count"),
        )
        .outerjoin(Author.posts)
        .group_by(Author.id, Author.name)
        .order_by(Author.name)
    )
    results = session.execute(stmt).all()

    return [{"name": name, "post_count": count} for name, count in results]
```

This executes a **single** query and uses the database's aggregation engine instead of loading all Post objects into Python.

**Task 2e: Verification Test**

```python
def test_report_includes_all_authors(engine):
    with Session(engine) as session:
        report = generate_author_report(session)

    names = {entry["name"] for entry in report}
    assert "Diana" in names, "Diana is missing from the report!"
    assert len(report) == 4, f"Expected 4 authors, got {len(report)}"

    diana_entry = next(e for e in report if e["name"] == "Diana")
    assert diana_entry["post_count"] == 0, "Diana should have 0 posts"

    alice_entry = next(e for e in report if e["name"] == "Alice")
    assert alice_entry["post_count"] == 3, "Alice should have 3 posts"
```

**The Key Insight:** `.join()` is a silent filter. It looks like it's connecting data, but it's also **removing** rows. Whenever your feature requires showing entities that might have zero related objects — empty carts, new users with no activity, categories with no products — you need `.outerjoin()`.

---

# LEVEL 2.5: EXTEND EXISTING CODE

## Exercise 2.5: The Blog Repository

**Given:** A working `PostRepository` class with a passing test suite. Your task is to add three features without breaking any existing tests.

### Provided Code (working and tested):

```python
# blog_repository.py

from sqlalchemy import create_engine, select, ForeignKey, String, Text, func
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, relationship,
    Session, selectinload
)
from datetime import datetime
from typing import Optional


class Base(DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "authors"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(200), unique=True)
    posts: Mapped[list["Post"]] = relationship(back_populates="author")
    def __repr__(self) -> str:
        return f"Author(id={self.id}, name='{self.name}')"


class Post(Base):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text)
    published_at: Mapped[datetime] = mapped_column()
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    author: Mapped["Author"] = relationship(back_populates="posts")
    def __repr__(self) -> str:
        return f"Post(id={self.id}, title='{self.title}')"


class PostRepository:
    def __init__(self, session: Session):
        self.session = session

    def get_all_posts(self) -> list[Post]:
        """Return all posts, newest first."""
        stmt = select(Post).order_by(Post.published_at.desc())
        return self.session.execute(stmt).scalars().all()

    def get_post_by_id(self, post_id: int) -> Optional[Post]:
        """Return a single post by ID, or None."""
        stmt = select(Post).where(Post.id == post_id)
        return self.session.execute(stmt).scalars().first()

    def get_posts_by_author(self, author_id: int) -> list[Post]:
        """Return all posts by a specific author, newest first."""
        stmt = (
            select(Post)
            .where(Post.author_id == author_id)
            .order_by(Post.published_at.desc())
        )
        return self.session.execute(stmt).scalars().all()

    def create_post(self, title: str, content: str, author_id: int) -> Post:
        """Create and return a new post."""
        post = Post(
            title=title,
            content=content,
            author_id=author_id,
            published_at=datetime.utcnow(),
        )
        self.session.add(post)
        self.session.flush()
        return post
```

### Provided Test Suite (all tests MUST continue to pass):

```python
# test_blog_repository.py

import pytest
from datetime import datetime, timedelta
from sqlalchemy import create_engine
from sqlalchemy.orm import Session
from blog_repository import Base, Author, Post, PostRepository


@pytest.fixture
def engine():
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    return engine


@pytest.fixture
def session(engine):
    with Session(engine) as session:
        yield session


@pytest.fixture
def seeded_session(session):
    alice = Author(name="Alice", email="alice@test.com")
    bob = Author(name="Bob", email="bob@test.com")
    session.add_all([alice, bob])
    session.flush()

    base = datetime(2025, 1, 1)
    posts = [
        Post(title="Post A", content="A", author=alice,
             published_at=base + timedelta(days=1)),
        Post(title="Post B", content="B", author=alice,
             published_at=base + timedelta(days=2)),
        Post(title="Post C", content="C", author=bob,
             published_at=base + timedelta(days=3)),
    ]
    session.add_all(posts)
    session.commit()
    return session


def test_get_all_posts(seeded_session):
    repo = PostRepository(seeded_session)
    posts = repo.get_all_posts()
    assert len(posts) == 3
    assert posts[0].title == "Post C"  # Newest first


def test_get_post_by_id(seeded_session):
    repo = PostRepository(seeded_session)
    post = repo.get_post_by_id(1)
    assert post is not None
    assert post.title == "Post A"


def test_get_post_by_id_not_found(seeded_session):
    repo = PostRepository(seeded_session)
    post = repo.get_post_by_id(999)
    assert post is None


def test_get_posts_by_author(seeded_session):
    repo = PostRepository(seeded_session)
    posts = repo.get_posts_by_author(1)  # Alice
    assert len(posts) == 2


def test_create_post(seeded_session):
    repo = PostRepository(seeded_session)
    post = repo.create_post("New Post", "Content", author_id=1)
    assert post.id is not None
    assert post.title == "New Post"
```

---

### Extension A: Add Tags (Many-to-Many)

Add a `Tag` model and a many-to-many relationship between `Post` and `Tag`. Then add these methods to `PostRepository`:

- `add_tag_to_post(post_id: int, tag_name: str) -> Post` — Adds a tag to a post. If the tag doesn't exist, create it. If it already exists, reuse it. Returns the updated post.
- `get_posts_by_tag(tag_name: str) -> list[Post]` — Returns all posts with the given tag, newest first.

**New tests that must pass:**

```python
def test_add_tag_to_post(seeded_session):
    repo = PostRepository(seeded_session)
    post = repo.add_tag_to_post(1, "python")
    assert any(t.name == "python" for t in post.tags)


def test_add_duplicate_tag(seeded_session):
    repo = PostRepository(seeded_session)
    repo.add_tag_to_post(1, "python")
    repo.add_tag_to_post(1, "python")  # Same tag again
    post = repo.get_post_by_id(1)
    python_tags = [t for t in post.tags if t.name == "python"]
    assert len(python_tags) == 1  # Should NOT duplicate


def test_get_posts_by_tag(seeded_session):
    repo = PostRepository(seeded_session)
    repo.add_tag_to_post(1, "python")
    repo.add_tag_to_post(2, "python")
    repo.add_tag_to_post(3, "database")

    python_posts = repo.get_posts_by_tag("python")
    assert len(python_posts) == 2

    db_posts = repo.get_posts_by_tag("database")
    assert len(db_posts) == 1


def test_get_posts_by_nonexistent_tag(seeded_session):
    repo = PostRepository(seeded_session)
    posts = repo.get_posts_by_tag("nonexistent")
    assert len(posts) == 0
```

**Constraint:** All original tests must still pass.

---

### Extension B: Eager Author Detail Endpoint

Add a method `get_author_detail(author_id: int) -> Optional[Author]` that returns an Author with their posts (and each post's tags) eagerly loaded. This method must execute **at most 3 queries** regardless of data size.

**New tests:**

```python
def test_author_detail_has_posts(seeded_session):
    repo = PostRepository(seeded_session)
    repo.add_tag_to_post(1, "python")
    repo.add_tag_to_post(1, "tutorial")

    author = repo.get_author_detail(1)  # Alice
    assert author is not None
    assert len(author.posts) == 2
    assert any("python" in [t.name for t in p.tags] for p in author.posts)


def test_author_detail_not_found(seeded_session):
    repo = PostRepository(seeded_session)
    author = repo.get_author_detail(999)
    assert author is None


def test_author_detail_no_detached_error(seeded_session):
    repo = PostRepository(seeded_session)
    author = repo.get_author_detail(1)
    seeded_session.close()  # Close the session!

    # This should NOT raise DetachedInstanceError because data was eagerly loaded
    assert len(author.posts) == 2
    for post in author.posts:
        _ = post.tags  # Should be accessible even with closed session
```

---

### Extension C: Keyset Pagination

Add a method `get_posts_paginated(cursor_id: Optional[int], page_size: int) -> list[Post]` that returns `page_size` posts using keyset pagination (ordered by `id` descending). If `cursor_id` is `None`, return the first page. Otherwise, return posts with `id < cursor_id`.

**New tests:**

```python
def test_first_page(seeded_session):
    repo = PostRepository(seeded_session)
    page = repo.get_posts_paginated(cursor_id=None, page_size=2)
    assert len(page) == 2
    assert page[0].id > page[1].id  # Descending order


def test_second_page(seeded_session):
    repo = PostRepository(seeded_session)
    page1 = repo.get_posts_paginated(cursor_id=None, page_size=2)
    cursor = page1[-1].id

    page2 = repo.get_posts_paginated(cursor_id=cursor, page_size=2)
    assert len(page2) == 1  # Only 3 posts total, page 1 had 2

    # No overlap
    page1_ids = {p.id for p in page1}
    page2_ids = {p.id for p in page2}
    assert len(page1_ids & page2_ids) == 0


def test_pagination_empty_page(seeded_session):
    repo = PostRepository(seeded_session)
    page = repo.get_posts_paginated(cursor_id=0, page_size=10)
    assert len(page) == 0  # No posts with id < 0
```

---

# Level 2.5 Solutions

## Solution: Extension A (Tags)

**Model additions:**

```python
from sqlalchemy import Table, Column

post_tags = Table(
    "post_tags",
    Base.metadata,
    Column("post_id", ForeignKey("posts.id"), primary_key=True),
    Column("tag_id", ForeignKey("tags.id"), primary_key=True),
)


class Tag(Base):
    __tablename__ = "tags"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)
    posts: Mapped[list["Post"]] = relationship(
        secondary=post_tags, back_populates="tags"
    )

# Add to Post class:
class Post(Base):
    # ... existing fields ...
    tags: Mapped[list["Tag"]] = relationship(
        secondary=post_tags, back_populates="posts"
    )
```

**Repository methods:**

```python
def add_tag_to_post(self, post_id: int, tag_name: str) -> Post:
    post = self.get_post_by_id(post_id)
    if post is None:
        raise ValueError(f"Post {post_id} not found")

    # Find or create the tag
    stmt = select(Tag).where(Tag.name == tag_name)
    tag = self.session.execute(stmt).scalars().first()
    if tag is None:
        tag = Tag(name=tag_name)
        self.session.add(tag)
        self.session.flush()

    # Only add if not already present
    if tag not in post.tags:
        post.tags.append(tag)
        self.session.flush()

    return post


def get_posts_by_tag(self, tag_name: str) -> list[Post]:
    stmt = (
        select(Post)
        .join(Post.tags)
        .where(Tag.name == tag_name)
        .order_by(Post.published_at.desc())
    )
    return self.session.execute(stmt).scalars().all()
```

**Why:** The `add_tag_to_post` method handles the duplicate check by using `if tag not in post.tags`. The `get_posts_by_tag` uses `.join(Post.tags)` which navigates through the junction table. This is an INNER JOIN, which is correct here — we only want posts that DO have the matching tag.

## Solution: Extension B (Eager Author Detail)

```python
def get_author_detail(self, author_id: int) -> Optional[Author]:
    stmt = (
        select(Author)
        .where(Author.id == author_id)
        .options(
            selectinload(Author.posts).selectinload(Post.tags)
        )
    )
    return self.session.execute(stmt).scalars().first()
```

**Why:** The chained `selectinload` generates at most 3 queries:
1. `SELECT * FROM authors WHERE id = ?`
2. `SELECT * FROM posts WHERE author_id IN (?)`
3. `SELECT tags.* FROM tags JOIN post_tags WHERE post_id IN (?)`

The `test_author_detail_no_detached_error` test verifies that eager loading happened — if it didn't, accessing `.posts` or `.tags` after session close would raise `DetachedInstanceError`.

## Solution: Extension C (Keyset Pagination)

```python
def get_posts_paginated(
    self, cursor_id: Optional[int], page_size: int
) -> list[Post]:
    stmt = select(Post).order_by(Post.id.desc()).limit(page_size)

    if cursor_id is not None:
        stmt = stmt.where(Post.id < cursor_id)

    return self.session.execute(stmt).scalars().all()
```

**Why:** With `cursor_id=None`, we get the newest `page_size` posts. With a cursor, we get posts older than the cursor. The `.id < cursor_id` combined with the index on the primary key means the database can jump directly to the right position — no scanning from row 0 like OFFSET.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## Exercise 3: Library Catalog API

**Scenario:** You're building the data access layer for an online library catalog. The catalog page displays books with their author and categories. The system must handle a library with 10,000 books, 2,000 authors, and 50 categories.

### Models (incomplete — you must add relationships):

```python
from sqlalchemy import (
    create_engine, select, ForeignKey, String, Text,
    Table, Column, func, and_, or_
)
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, relationship,
    Session, selectinload, joinedload
)
from typing import Optional


class Base(DeclarativeBase):
    pass


book_categories = Table(
    "book_categories",
    Base.metadata,
    Column("book_id", ForeignKey("books.id"), primary_key=True),
    Column("category_id", ForeignKey("categories.id"), primary_key=True),
)


class Author(Base):
    __tablename__ = "authors"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    # TODO: Define relationship to books


class Book(Base):
    __tablename__ = "books"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(300))
    published_year: Mapped[int] = mapped_column()
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    # TODO: Define relationship to author
    # TODO: Define relationship to categories


class Category(Base):
    __tablename__ = "categories"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100), unique=True)
    # TODO: Define relationship to books
```

### Seed Data:

```python
def create_library_database():
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)

    with Session(engine) as session:
        # Authors — note: some have NO books
        authors = [
            Author(name="Alice Author"),     # Will have 3 books
            Author(name="Bob Writer"),       # Will have 2 books
            Author(name="Charlie Novelist"), # Will have 0 books ← IMPORTANT
        ]
        session.add_all(authors)
        session.flush()

        # Categories
        fiction = Category(name="Fiction")
        science = Category(name="Science")
        tutorial = Category(name="Tutorial")
        session.add_all([fiction, science, tutorial])
        session.flush()

        # Books
        books_data = [
            ("Python Mastery", 2024, authors[0], [tutorial, science]),
            ("Async Patterns", 2023, authors[0], [tutorial]),
            ("The ORM Novel", 2025, authors[0], [fiction]),
            ("SQL Deep Dive", 2024, authors[1], [tutorial, science]),
            ("Database Stories", 2023, authors[1], [fiction, science]),
        ]
        for title, year, author, cats in books_data:
            book = Book(title=title, published_year=year, author=author)
            book.categories = cats
            session.add(book)

        session.commit()

    return engine
```

### Constraints:

1. **Query budget:** The book listing function must execute **at most 3 SQL queries** for any number of books, authors, and categories.
2. **Pagination stability:** Paginated results must have **zero duplicates** across pages, even if books are inserted between requests.
3. **Inclusivity:** The author listing function must include **all** authors, including those with zero books.

---

### Task 3a: Define Models and Implement Basic Listing

Complete the model definitions with proper relationships. Then implement:

```python
def get_books_with_details(session: Session) -> list[dict]:
    """
    Return all books with author name and category names.
    Must execute at most 3 queries.
    """
    pass
```

**Tests:**

```python
def test_books_with_details_correctness(engine):
    with Session(engine) as session:
        books = get_books_with_details(session)

    assert len(books) == 5
    python_book = next(b for b in books if b["title"] == "Python Mastery")
    assert python_book["author"] == "Alice Author"
    assert set(python_book["categories"]) == {"Tutorial", "Science"}


def test_books_with_details_query_budget(engine):
    from sqlalchemy import event

    counter = QueryCounter(engine)
    counter.reset()

    with Session(engine) as session:
        books = get_books_with_details(session)

    assert counter.count <= 3, (
        f"Query budget exceeded: {counter.count} queries (max 3)"
    )
```

---

### Task 3b: Implement Keyset Pagination

```python
def get_books_page(
    session: Session,
    cursor_id: Optional[int] = None,
    page_size: int = 2,
) -> list[dict]:
    """
    Return a page of books using keyset pagination (by id DESC).
    Must include author and categories.
    Must have zero duplicates across pages.
    """
    pass
```

**Tests:**

```python
def test_pagination_no_duplicates(engine):
    with Session(engine) as session:
        page1 = get_books_page(session, cursor_id=None, page_size=2)
        assert len(page1) == 2

        cursor = page1[-1]["id"]
        page2 = get_books_page(session, cursor_id=cursor, page_size=2)
        assert len(page2) == 2

        cursor = page2[-1]["id"]
        page3 = get_books_page(session, cursor_id=cursor, page_size=2)
        assert len(page3) == 1  # Only 5 books total

        all_ids = (
            [b["id"] for b in page1]
            + [b["id"] for b in page2]
            + [b["id"] for b in page3]
        )
        assert len(all_ids) == len(set(all_ids)), "Duplicates found!"
        assert len(all_ids) == 5


def test_pagination_stable_under_insert(engine):
    with Session(engine) as session:
        page1 = get_books_page(session, cursor_id=None, page_size=2)
        cursor = page1[-1]["id"]

    # Insert a new book between page requests
    with Session(engine) as session:
        author = session.execute(select(Author).limit(1)).scalars().first()
        new_book = Book(title="New Release", published_year=2025, author=author)
        session.add(new_book)
        session.commit()

    with Session(engine) as session:
        page2 = get_books_page(session, cursor_id=cursor, page_size=2)

    page1_ids = {b["id"] for b in page1}
    page2_ids = {b["id"] for b in page2}
    assert len(page1_ids & page2_ids) == 0, "Duplicate across pages after insert!"
```

---

### Task 3c: Author Listing with Stats

```python
def get_all_authors_with_stats(session: Session) -> list[dict]:
    """
    Return ALL authors with their book count.
    Authors with 0 books MUST be included.
    Must NOT use N+1 queries.
    """
    pass
```

**Tests:**

```python
def test_all_authors_included(engine):
    with Session(engine) as session:
        authors = get_all_authors_with_stats(session)

    names = {a["name"] for a in authors}
    assert "Charlie Novelist" in names, "Author with 0 books is missing!"
    assert len(authors) == 3


def test_author_stats_correct(engine):
    with Session(engine) as session:
        authors = get_all_authors_with_stats(session)

    alice = next(a for a in authors if a["name"] == "Alice Author")
    assert alice["book_count"] == 3

    charlie = next(a for a in authors if a["name"] == "Charlie Novelist")
    assert charlie["book_count"] == 0


def test_author_stats_query_efficiency(engine):
    counter = QueryCounter(engine)
    counter.reset()

    with Session(engine) as session:
        authors = get_all_authors_with_stats(session)

    # Should be 1 query (aggregation in SQL), or at most 2 (with eager load)
    assert counter.count <= 2, (
        f"Too many queries: {counter.count} (expected ≤ 2)"
    )
```

---

# Level 3 Solutions

## Solution 3a: Models and Book Listing

**Model definitions:**

```python
class Author(Base):
    __tablename__ = "authors"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    books: Mapped[list["Book"]] = relationship(back_populates="author")


class Book(Base):
    __tablename__ = "books"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(300))
    published_year: Mapped[int] = mapped_column()
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    author: Mapped["Author"] = relationship(back_populates="books")
    categories: Mapped[list["Category"]] = relationship(
        secondary=book_categories, back_populates="books"
    )


class Category(Base):
    __tablename__ = "categories"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100), unique=True)
    books: Mapped[list["Book"]] = relationship(
        secondary=book_categories, back_populates="categories"
    )
```

**Book listing:**

```python
def get_books_with_details(session: Session) -> list[dict]:
    stmt = (
        select(Book)
        .options(
            selectinload(Book.author),
            selectinload(Book.categories),
        )
        .order_by(Book.id)
    )
    books = session.execute(stmt).scalars().all()

    return [
        {
            "id": book.id,
            "title": book.title,
            "published_year": book.published_year,
            "author": book.author.name,
            "categories": [c.name for c in book.categories],
        }
        for book in books
    ]
```

**Why 3 queries:** `selectinload(Book.author)` fires a second query loading all relevant authors via `WHERE id IN (...)`. `selectinload(Book.categories)` fires a third query loading all categories through the junction table. Regardless of data size, always 3 queries.

**Why NOT `joinedload` for both?** Using `joinedload(Book.author)` AND `joinedload(Book.categories)` on the same query would cause a cartesian product. A book with 1 author and 3 categories would produce 3 rows. `selectinload` avoids this entirely by using separate `IN` queries.

## Solution 3b: Keyset Pagination

```python
def get_books_page(
    session: Session,
    cursor_id: Optional[int] = None,
    page_size: int = 2,
) -> list[dict]:
    stmt = (
        select(Book)
        .options(
            selectinload(Book.author),
            selectinload(Book.categories),
        )
        .order_by(Book.id.desc())
        .limit(page_size)
    )

    if cursor_id is not None:
        stmt = stmt.where(Book.id < cursor_id)

    books = session.execute(stmt).scalars().all()

    return [
        {
            "id": book.id,
            "title": book.title,
            "author": book.author.name,
            "categories": [c.name for c in book.categories],
        }
        for book in books
    ]
```

**Why this resists inserts:** The cursor (`Book.id < cursor_id`) anchors to a value, not a position. Inserting a new book with a higher ID doesn't affect the `WHERE id < 5` clause — it still returns the same books below ID 5.

## Solution 3c: Author Stats

```python
def get_all_authors_with_stats(session: Session) -> list[dict]:
    stmt = (
        select(
            Author.name,
            func.count(Book.id).label("book_count"),
        )
        .outerjoin(Author.books)       # LEFT OUTER JOIN — keeps Charlie
        .group_by(Author.id, Author.name)
        .order_by(Author.name)
    )
    results = session.execute(stmt).all()

    return [
        {"name": name, "book_count": count}
        for name, count in results
    ]
```

**Why `.outerjoin()`:** An INNER JOIN would exclude Charlie (0 books). **Why `func.count(Book.id)` instead of `func.count()`:** `COUNT(books.id)` counts non-NULL book IDs. For Charlie, the LEFT JOIN produces a row with `books.id = NULL`, and `COUNT(NULL) = 0`. If we used `COUNT(*)`, Charlie would get 1 (counting the row itself).

**This is a single query** — the database does the aggregation, not Python. No N+1.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## Exercise 4: When the Cure Is Worse Than the Disease

A developer reads about the N+1 problem and decides: "I'll **always** use `selectinload` on **every** relationship in **every** query. That way I'll never have N+1."

They apply this philosophy to a user profile endpoint:

```python
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(50))
    posts: Mapped[list["Post"]] = relationship(back_populates="user")
    comments: Mapped[list["Comment"]] = relationship(back_populates="user")
    notifications: Mapped[list["Notification"]] = relationship(back_populates="user")
    audit_logs: Mapped[list["AuditLog"]] = relationship(back_populates="user")

# The "always eager load everything" approach:
def get_user_profile(session: Session, user_id: int) -> dict:
    stmt = (
        select(User)
        .where(User.id == user_id)
        .options(
            selectinload(User.posts),          # User has 500 posts
            selectinload(User.comments),       # User has 2,000 comments
            selectinload(User.notifications),  # User has 10,000 notifications
            selectinload(User.audit_logs),     # User has 50,000 audit logs
        )
    )
    user = session.execute(stmt).scalars().first()
    
    # But the profile page only shows:
    return {
        "username": user.username,
        "post_count": len(user.posts),  # Only needs the COUNT, not all 500 objects
    }
```

**Task:**

1. **Describe the problem.** The developer has zero N+1 queries. But the endpoint is still slow and uses enormous memory. Why? Calculate the approximate number of Python objects created.

2. **Write a counter-example.** Produce a version of `get_user_profile` that is **faster** and uses **less memory** than the "eagerly load everything" version. Explain why fewer queries is NOT the only performance metric.

3. **State the general principle.** In what scenarios is lazy loading **actually the correct choice**? When should you avoid eager loading?

---

# Level 4 Solution

**1. The problem:**

The developer loads **62,500 Python objects** (500 posts + 2,000 comments + 10,000 notifications + 50,000 audit logs) into memory just to display a profile page that only needs a **username** and a **count**. The 5 queries execute quickly, but:

- The database must serialize 62,500 rows across the network
- SQLAlchemy must construct 62,500 Python objects (each with attribute overhead)
- Python's memory footprint for this single request could exceed 100MB
- The garbage collector must eventually clean all of it up

Zero N+1 queries, but massive bandwidth waste and memory pressure.

**2. The counter-example:**

```python
def get_user_profile(session: Session, user_id: int) -> dict:
    # Query 1: Just the user (no relationships loaded)
    stmt = select(User).where(User.id == user_id)
    user = session.execute(stmt).scalars().first()
    
    # Query 2: Count posts in the DATABASE, not in Python
    post_count = session.execute(
        select(func.count(Post.id)).where(Post.user_id == user_id)
    ).scalar()
    
    return {
        "username": user.username,
        "post_count": post_count,
    }
```

This executes **2 queries** (more than 1!), loads **1 Python object** (just the User), and the database computes the count using an index — near-instant.

The "always eager load" version did 5 queries but transferred 62,500 rows. This version does 2 queries and transfers 2 rows. **Fewer queries is not always faster.** The total work (network + memory + processing) is what matters.

**3. The general principle:**

Lazy loading is the correct choice when:
- **You might not access the relationship at all** (conditional logic paths)
- **You only need a single object's relationship** (not in a loop — no N+1 risk)
- **The relationship has many objects but you only need an aggregate** (use `func.count()` instead)
- **The endpoint is performance-critical and you want to minimize data transfer**

Eager loading is the wrong choice when:
- You load data you never use (wasted bandwidth and memory)
- The relationship has thousands or millions of rows (don't load 50,000 audit logs into Python)
- You only need a count, a max, or an existence check (use SQL aggregates instead)

**The meta-insight:** The N+1 problem is about queries **in a loop**. If you're accessing a relationship on a single object, outside a loop, lazy loading fires exactly 1 query — which is fine. The "always eager load" dogma trades N+1 for a different problem: loading data you never use.

---

# LEVEL 5: THE TEACH BACK

## Exercise 5: "One Query to Rule Them All"

**Scenario:** A junior developer on your team submits a PR where every query uses `joinedload` for all relationships. In the PR description, they write:

> "I refactored all queries to use `joinedload`. Since `joinedload` uses 1 query and `selectinload` uses 2 queries, `joinedload` is always better. Fewer round-trips to the database = faster performance. I also removed all `.unique()` calls because they seemed redundant."

Their code includes queries like:

```python
# From the PR:
stmt = (
    select(Author)
    .options(
        joinedload(Author.posts),
        joinedload(Author.comments),
    )
)
authors = session.execute(stmt).scalars().all()
```

**Your task:**

1. **Identify every incorrect claim** in the PR description. Explain why each is wrong.

2. **Explain the Cartesian product problem** that occurs when using multiple `joinedload` on the same query. If an author has 10 posts and 5 comments, how many rows does the SQL result contain? What does this do to network bandwidth and deduplication cost?

3. **Explain why removing `.unique()` breaks the code.** What specific bug will this cause in the `authors` list?

4. **Write the code review comment** you would leave on this PR. Be specific, cite the failure modes, and suggest the correct approach. Your comment should educate, not just reject.

---

# Level 5 Solution

**1. Incorrect claims:**

**Claim: "joinedload uses 1 query and selectinload uses 2, so joinedload is always better."**
Fewer queries does not mean less total work. `joinedload` produces a single query with a LEFT OUTER JOIN, which **duplicates parent rows** — one row per child. If Alice has 100 posts, Alice's data (name, email, etc.) is transmitted 100 times across the network. `selectinload` sends 2 smaller queries with zero duplication. For large relationships, `selectinload` transfers **less total data** despite using more queries.

**Claim: ".unique() seemed redundant."**
`.unique()` is mandatory with `joinedload`. Without it, the result list contains duplicate references to the same parent object — one per child row. Removing it silently corrupts counts, iterations, and any logic that assumes the list contains unique entities.

**2. The Cartesian product:**

When you apply `joinedload` to two independent relationships on the same parent, the SQL produces a **CROSS JOIN** between them:

```sql
SELECT authors.*, posts.*, comments.*
FROM authors
LEFT JOIN posts ON authors.id = posts.author_id
LEFT JOIN comments ON authors.id = comments.author_id
```

For an author with 10 posts and 5 comments, this produces **10 × 5 = 50 rows** — not 15 (10 + 5). Each post is paired with each comment. This is a cartesian product:

| author | post | comment |
|--------|------|---------|
| Alice | Post 1 | Comment 1 |
| Alice | Post 1 | Comment 2 |
| Alice | Post 1 | Comment 3 |
| ... | ... | ... |
| Alice | Post 10 | Comment 5 |

50 rows instead of 15. With 100 posts and 50 comments, that's **5,000 rows** per author. The data duplication is quadratic, not linear.

**3. Why removing `.unique()` breaks the code:**

Without `.unique()`, `session.execute(stmt).scalars().all()` returns one list entry per JOIN result row. For Alice with 10 posts and 5 comments (50 cartesian rows), Alice appears **50 times** in the list. `len(authors)` returns the wrong count. Iterating over the list processes Alice 50 times. Any aggregate or display logic is silently corrupted.

**4. Code review comment:**

> Hey — thanks for tackling the N+1 issue! That's a real production concern and it's great you're thinking about it. However, this approach introduces two new problems:
>
> **Problem 1: Cartesian product.** Using multiple `joinedload` calls on the same query creates a SQL CROSS JOIN between the relationships. With 10 posts and 5 comments per author, you get 50 rows per author instead of 15. This actually **increases** network traffic and processing cost. Use `selectinload` when loading multiple independent relationships on the same parent.
>
> **Problem 2: Missing `.unique()`.** `joinedload` duplicates parent rows in the SQL result. Without `.unique()`, the Python list contains the same Author object repeated once per child. This means `len(authors)` returns the wrong count, and loops process each author multiple times.
>
> **Suggested fix:**
> ```python
> stmt = (
>     select(Author)
>     .options(
>         selectinload(Author.posts),     # 2 queries, no duplication
>         selectinload(Author.comments),  # 3 queries total, no cartesian product
>     )
> )
> authors = session.execute(stmt).scalars().all()  # No .unique() needed
> ```
>
> **Rule of thumb:** Use `joinedload` only when loading a single small relationship (≤20 related objects) where you need exactly 1 DB round-trip. For everything else — especially multiple relationships or large collections — `selectinload` is the safer default.