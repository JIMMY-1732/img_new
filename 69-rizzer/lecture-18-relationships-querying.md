# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PAIN FIRST, MAGIC SECOND                                       │
│  ────────────────────────                                       │
│  Students will try to get related data WITHOUT relationships.   │
│  They'll write tedious manual queries. Then we give them        │
│  relationship(). The relief IS the understanding.               │
│                                                                 │
│  EVERY ORM LINE HAS A SQL SHADOW                                │
│  ────────────────────────────────                               │
│  Students learned raw SQL in Week 5. Every SQLAlchemy call,     │
│  we show the SQL it generates. No magic. No mystery.            │
│  The ORM is a translator, not a black box.                      │
│                                                                 │
│  THE N+1 PROBLEM IS THE LECTURE'S CORE                          │
│  ─────────────────────────────────────                          │
│  This is the single most common ORM performance mistake in      │
│  production. We will make them SEE every wasted query.          │
│  echo=True is our flashlight in the dark.                       │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Week 5 SQL → "You already wrote these JOINs by hand"           │
│  Week 6 L1 → "Building on models you defined last lecture"      │
│  Week 1 L1 → "Mapped[list['Post']] uses generics/type hints"   │
│  Week 4 L2 → "Pagination concepts from API design, now in DB"  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                  RELATIONSHIPS & QUERYING                       │
│                     (3-4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (30 min)                                   │
│  ├─ 1.1 Models Without Relationships (The Disconnected World)   │
│  ├─ 1.2 The Manual Way (Feel the Pain)                          │
│  ├─ 1.3 What We Actually Want                                   │
│  └─ 1.4 Relationship Types Recap (Week 5 → ORM)                │
│                                                                 │
│  PART 2: DEFINING RELATIONSHIPS (60 min)                        │
│  ├─ 2.1 One-to-Many: Foreign Keys in SQLAlchemy                 │
│  ├─ 2.2 relationship() — The Bridge Between Models              │
│  ├─ 2.3 back_populates — The Two-Way Street                     │
│  ├─ 2.4 Creating and Accessing Related Objects                  │
│  ├─ 2.5 Many-to-Many: The Association Table                     │
│  └─ 2.6 Working with Many-to-Many                               │
│                                                                 │
│  PART 3: QUERYING (50 min)                                      │
│  ├─ 3.1 select() — The Foundation (SQLAlchemy 2.0)              │
│  ├─ 3.2 where() — Filtering Results                             │
│  ├─ 3.3 Combining Filters (and_, or_, in_)                      │
│  ├─ 3.4 Ordering and Limiting (order_by, limit)                 │
│  └─ 3.5 Querying Across Relationships (join)                    │
│                                                                 │
│  PART 4: THE N+1 PROBLEM (45 min)                               │
│  ├─ 4.1 The Phone Call Problem (Demonstration)                  │
│  ├─ 4.2 Lazy Loading (The Default Trap)                         │
│  ├─ 4.3 joinedload() — One Big Query                            │
│  ├─ 4.4 selectinload() — Two Smart Queries                      │
│  └─ 4.5 Choosing Your Loading Strategy                          │
│                                                                 │
│  PART 5: PAGINATION (30 min)                                    │
│  ├─ 5.1 Offset-Based Pagination                                 │
│  ├─ 5.2 Why Offset Breaks at Scale                              │
│  ├─ 5.3 Keyset-Based (Cursor) Pagination                        │
│  └─ 5.4 The Decision Framework                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## IMPORTS REFERENCE

**Everything used in this lecture, in one place:**

```python
# SQLAlchemy core
from sqlalchemy import (
    create_engine, select, ForeignKey, String, Text,
    Table, Column, and_, or_, func
)

# SQLAlchemy ORM
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, relationship,
    Session, joinedload, selectinload
)

# Standard library
from datetime import datetime
from typing import Optional
```

---

# PART 1: THE PROBLEM

## 1.1 Models Without Relationships (The Disconnected World)

**Start where Lecture 1 left off. Recall what students can already build:**

```python
# What you know from Lecture 1 — basic models, no relationships

from sqlalchemy import create_engine, String, Text, ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(200), unique=True)

class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text)
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    #          ▲
    #          You know this from Week 5 SQL.
    #          It's the foreign key column. A real column in the database.
    #          But right now, that's ALL it is — a number sitting in a column.
```

**These models are *disconnected* in Python.** The `author_id` column stores a number, but Python doesn't know that number *means* anything. There's no bridge between the `Author` object and the `Post` object.

```
┌─────────────────────────────────────────────────────────────────┐
│              THE DISCONNECTED WORLD                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  In the DATABASE (Week 5):                                      │
│                                                                 │
│  authors table              posts table                         │
│  ┌────┬─────────┐           ┌────┬──────────┬───────────┐       │
│  │ id │ name    │           │ id │ title    │ author_id │       │
│  ├────┼─────────┤           ├────┼──────────┼───────────┤       │
│  │ 1  │ Alice   │◄──────────│ 1  │ Hello    │     1     │       │
│  │ 2  │ Bob     │◄──────┐   │ 2  │ Async    │     1     │       │
│  └────┴─────────┘       │   │ 3  │ REST API │     2     │       │
│                         └───│ 4  │ Testing  │     2     │       │
│  The foreign key ────────▶  └────┴──────────┴───────────┘       │
│  connects them in SQL.                                          │
│                                                                 │
│                                                                 │
│  In PYTHON (right now):                                         │
│                                                                 │
│   Author(id=1, name="Alice")       Post(id=1, author_id=1)     │
│                                                                 │
│          ???                               ???                  │
│                                                                 │
│   No connection. Two separate objects.                          │
│   author_id=1 is just a number. Python doesn't "know"          │
│   that it points to Alice.                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.2 The Manual Way (Feel the Pain)

**Let's try to work with related data using only what we know so far.**

```python
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

engine = create_engine("postgresql://user:pass@localhost/blog", echo=True)

with Session(engine) as session:

    # TASK: Get Alice and all her posts.

    # Step 1: Find Alice
    stmt = select(Author).where(Author.name == "Alice")
    alice = session.execute(stmt).scalars().first()

    # Step 2: Now find her posts — SEPARATE query, manual foreign key
    stmt = select(Post).where(Post.author_id == alice.id)
    alice_posts = session.execute(stmt).scalars().all()

    print(f"Alice's posts: {[p.title for p in alice_posts]}")

    # TASK: Given a post, find who wrote it.

    some_post = alice_posts[0]
    # some_post.author_id is 1... but that's just a number.
    # To get the AUTHOR OBJECT, we need ANOTHER query:
    stmt = select(Author).where(Author.id == some_post.author_id)
    author = session.execute(stmt).scalars().first()

    print(f"Post '{some_post.title}' was written by {author.name}")
```

**Now ask the class:**

> "That works. But what's wrong with it? Imagine you're building an API endpoint that returns 50 posts, each with the author's name. How many queries do you need?"

Answer: **51 queries.** 1 for the posts, then 50 more to look up each author. Every single time.

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE PAIN POINTS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. TEDIOUS: Two separate queries just to get author + posts    │
│                                                                 │
│  2. ERROR-PRONE: You manage foreign keys manually.              │
│     Forget to match author_id? Silent bug.                      │
│                                                                 │
│  3. NOT PYTHONIC: You're working with IDs, not objects.          │
│     post.author_id is a number. You WANT post.author.           │
│                                                                 │
│  4. SCALES TERRIBLY: 50 posts = 51 queries.                     │
│     500 posts = 501 queries. This kills your server.            │
│                                                                 │
│  In Week 5, you wrote JOINs to solve this in SQL.               │
│  But right now, SQLAlchemy doesn't know your models             │
│  are RELATED. It can't JOIN for you.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 What We Actually Want

**This is what relationship() will give us:**

```python
# AFTER this lecture — what your code will look like:

with Session(engine) as session:

    # Get Alice
    stmt = select(Author).where(Author.name == "Alice")
    alice = session.execute(stmt).scalars().first()

    # Get her posts — just ACCESS THE ATTRIBUTE
    print(alice.posts)          # → [Post(...), Post(...), ...]
    print(alice.posts[0].title) # → "Hello World"

    # From a post, get the author — just ACCESS THE ATTRIBUTE
    post = alice.posts[0]
    print(post.author)          # → Author(name="Alice", ...)
    print(post.author.name)     # → "Alice"

    # No manual queries. No foreign key juggling.
    # Python objects connected like a graph.
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  BEFORE (disconnected):          AFTER (relationships):         │
│                                                                 │
│  alice = Author(id=1)            alice = Author(id=1)           │
│  alice.???                       alice.posts → [Post, Post]     │
│                                      │                          │
│  post = Post(author_id=1)        post = Post(author_id=1)       │
│  post.???                        post.author → Author("Alice")  │
│                                      │                          │
│                                  Two-way navigation.            │
│                                  Python objects, not IDs.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 Relationship Types Recap (Week 5 → ORM)

**You designed these in Week 5's Database Design Workshop. Quick recap:**

```
┌─────────────────────────────────────────────────────────────────┐
│              RELATIONSHIP TYPES (Week 5 Recap)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ONE-TO-MANY                                                    │
│  ────────────                                                   │
│  One author writes MANY posts.                                  │
│  One post belongs to ONE author.                                │
│                                                                 │
│    Author ──────< Post                                          │
│     (one)         (many)                                        │
│                                                                 │
│  FK lives on the MANY side: posts.author_id → authors.id        │
│                                                                 │
│                                                                 │
│  MANY-TO-MANY                                                   │
│  ─────────────                                                  │
│  One post can have MANY tags.                                   │
│  One tag can be on MANY posts.                                  │
│                                                                 │
│    Post >──────< Tag                                            │
│    (many)        (many)                                         │
│                                                                 │
│  Requires a JUNCTION TABLE: post_tags(post_id, tag_id)          │
│  You built these in Week 5, Lecture 4.                          │
│                                                                 │
│                                                                 │
│  ONE-TO-ONE (less common)                                       │
│  ────────────────────────                                       │
│  One user has ONE profile. One profile belongs to ONE user.     │
│  Same as one-to-many, but with a unique constraint on the FK.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "In Week 5, you expressed these relationships in SQL with `FOREIGN KEY` and `JOIN`. Now we'll express them in Python with `ForeignKey` and `relationship()`. Same concepts, new syntax."

---

# PART 2: DEFINING RELATIONSHIPS

## 2.1 One-to-Many: Foreign Keys in SQLAlchemy

**The foreign key column is the physical connection in the database. You already know this from Week 5.**

```python
class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text)

    # The foreign key — a REAL column in the database
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    #                                       ▲
    #                    Points to the TABLE NAME and COLUMN NAME
    #                    Not the Python class name!
    #                    "authors.id" = authors table, id column
```

**This is what you already know from Lecture 1.** The `ForeignKey("authors.id")` creates the same constraint you wrote in Week 5:

```sql
-- What this becomes in the database (Week 5 SQL)
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,
    author_id INTEGER NOT NULL REFERENCES authors(id)
    --                                  ▲
    --                     Same thing, SQL version
);
```

**But the foreign key alone doesn't give us `post.author` or `alice.posts`.** It's just a number in a column. We need the bridge.

---

## 2.2 relationship() — The Bridge Between Models

**`relationship()` creates a Python-level property that lets you navigate between related objects.**

```python
from sqlalchemy.orm import relationship

class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(200), unique=True)

    # NEW: relationship property — NOT a database column!
    posts: Mapped[list["Post"]] = relationship()
    #      ▲                       ▲
    #      │                       └─ Creates the bridge
    #      │
    #      └─ Type hint tells Python AND you:
    #         "This is a LIST of Post objects"
    #         (Uses generics from Week 1, Lecture 1!)
```

```
┌─────────────────────────────────────────────────────────────────┐
│            FOREIGN KEY vs RELATIONSHIP                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ForeignKey COLUMN (author_id):                                 │
│  ──────────────────────────────                                 │
│  • Lives in the DATABASE (it's a real column)                   │
│  • Stores the actual value (e.g., author_id = 1)               │
│  • Enforces referential integrity                               │
│  • You defined these in Week 5 SQL                              │
│                                                                 │
│  relationship() PROPERTY (posts, author):                       │
│  ────────────────────────────────────────                       │
│  • Lives in PYTHON ONLY (not a database column)                 │
│  • Gives you convenient access to related objects               │
│  • alice.posts → [Post(...), Post(...)]                         │
│  • post.author → Author(name="Alice")                           │
│  • SQLAlchemy uses the ForeignKey to figure out the query       │
│                                                                 │
│  You need BOTH:                                                 │
│  ┌────────────────┐       ┌─────────────────────┐               │
│  │ ForeignKey     │       │ relationship()      │               │
│  │ = the physical │  ───▶ │ = the Python        │               │
│  │   wire in the  │       │   shortcut to       │               │
│  │   database     │       │   navigate it       │               │
│  └────────────────┘       └─────────────────────┘               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How does SQLAlchemy know which foreign key to use?**

> "When you write `posts: Mapped[list["Post"]] = relationship()`, SQLAlchemy looks at the `Post` model, finds a `ForeignKey("authors.id")`, and says: *'That points back to Author. That's the connection.'* It figures out the link automatically."

```
┌─────────────────────────────────────────────────────────────────┐
│         HOW SQLAlchemy RESOLVES THE RELATIONSHIP                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Author model has:                                              │
│    posts: Mapped[list["Post"]] = relationship()                 │
│                         │                                       │
│                         ▼                                       │
│  SQLAlchemy looks at Post model...                              │
│                         │                                       │
│                         ▼                                       │
│  Finds: author_id = mapped_column(ForeignKey("authors.id"))     │
│                                                │                │
│                                                ▼                │
│  "authors.id" matches the Author table!                         │
│                                                │                │
│                                                ▼                │
│  Connection resolved: Author.posts uses Post.author_id          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 back_populates — The Two-Way Street

**Right now, we can go from Author → Posts. But what about Post → Author?**

```python
# One-way relationship (incomplete)
class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(200), unique=True)

    posts: Mapped[list["Post"]] = relationship()  # Author → Posts ✅

class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text)
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))

    # Post → Author? ❌ No way to do post.author yet!
```

**`back_populates` creates the two-way link:**

```python
class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(200), unique=True)

    # "My 'posts' property mirrors Post's 'author' property"
    posts: Mapped[list["Post"]] = relationship(back_populates="author")
    #                                                         ▲
    #                          Name of the ATTRIBUTE on the Post class


class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text)
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))

    # "My 'author' property mirrors Author's 'posts' property"
    author: Mapped["Author"] = relationship(back_populates="posts")
    #       ▲                                                ▲
    #       │                              Name of the ATTRIBUTE on Author
    #       │
    #       └── Not a list! One post has ONE author.
    #           Mapped["Author"], not Mapped[list["Author"]]
```

```
┌─────────────────────────────────────────────────────────────────┐
│              back_populates — THE TWO-WAY STREET                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│            Author                          Post                 │
│  ┌─────────────────────┐        ┌────────────────────────┐      │
│  │ id                  │        │ id                     │      │
│  │ name                │        │ title                  │      │
│  │ email               │        │ content                │      │
│  │                     │        │ author_id (ForeignKey) │      │
│  │ posts ──────────────────────▶│                        │      │
│  │   Mapped[list[Post]]│   back │ author ◀──────────────┐│      │
│  │                     │◀───────│   Mapped[Author]      ││      │
│  └─────────────────────┘ populates └──────────────────────┘     │
│                                                                 │
│  alice.posts → [Post("Hello"), Post("Async")]                   │
│  post.author → Author("Alice")                                  │
│                                                                 │
│  They MUST point to each other's attribute name:                │
│  Author.posts  ← back_populates="author"  (Post's attribute)   │
│  Post.author   ← back_populates="posts"   (Author's attribute) │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The type hints tell you the cardinality:**

```python
# ONE-to-MANY side (the "one" has a list):
posts: Mapped[list["Post"]] = relationship(back_populates="author")
#             ▲
#             list = "many posts"

# MANY-to-ONE side (the "many" has a single object):
author: Mapped["Author"] = relationship(back_populates="posts")
#              ▲
#              No list = "one author"

# OPTIONAL relationship (author might be None):
author: Mapped[Optional["Author"]] = relationship(back_populates="posts")
#              ▲
#              Optional from Week 1, Lecture 1!
```

> "Notice the forward reference strings: `"Post"` and `"Author"` are in quotes. This is because when Python processes the `Author` class, the `Post` class hasn't been defined yet. The string tells SQLAlchemy: *'I'll look up this class later, after all models are defined.'* You've seen this with type hints in Lecture 1."

---

## 2.4 Creating and Accessing Related Objects

**Now let's see relationships in action:**

```python
from sqlalchemy.orm import Session

with Session(engine) as session:

    # Create an author
    alice = Author(name="Alice", email="alice@example.com")

    # Create posts — assign the RELATIONSHIP, not the foreign key
    post1 = Post(title="Hello World", content="My first post", author=alice)
    post2 = Post(title="Async Deep Dive", content="Understanding await", author=alice)
    #                                                                      ▲
    #                                              Assign the Author OBJECT,
    #                                              not author_id=1!
    #                                              SQLAlchemy handles the FK.

    session.add(alice)  # Alice + her posts are all added (cascade)
    session.commit()
```

**The SQL generated (echo=True):**

```sql
-- SQLAlchemy generates this for you:
INSERT INTO authors (name, email) VALUES ('Alice', 'alice@example.com') RETURNING id
-- Alice gets id=1

INSERT INTO posts (title, content, author_id) VALUES ('Hello World', 'My first post', 1)
INSERT INTO posts (title, content, author_id) VALUES ('Async Deep Dive', 'Understanding await', 1)
--                                                     ▲
--                                     SQLAlchemy filled in author_id=1 automatically!
```

**Alternative: append to the list side**

```python
with Session(engine) as session:

    bob = Author(name="Bob", email="bob@example.com")

    # Create posts separately, then append to the author's list
    post3 = Post(title="REST API Design", content="Best practices")
    post4 = Post(title="Testing Strategies", content="pytest patterns")

    bob.posts.append(post3)  # Add to the relationship list
    bob.posts.append(post4)

    session.add(bob)  # Bob + his posts are all added
    session.commit()
```

**Both approaches work. SQLAlchemy keeps them in sync:**

```python
with Session(engine) as session:

    stmt = select(Author).where(Author.name == "Alice")
    alice = session.execute(stmt).scalars().first()

    # Navigate Author → Posts
    for post in alice.posts:
        print(f"  {post.title}")
    # Output:
    #   Hello World
    #   Async Deep Dive

    # Navigate Post → Author
    stmt = select(Post).where(Post.title == "Hello World")
    post = session.execute(stmt).scalars().first()

    print(post.author.name)  # → "Alice"

    # back_populates keeps both sides consistent:
    new_post = Post(title="New Post", content="Fresh content", author=alice)
    session.add(new_post)

    print(new_post.author.name)  # → "Alice"  (Post → Author)
    print(new_post in alice.posts)  # → True   (Author → Posts updated too!)
```

```
┌─────────────────────────────────────────────────────────────────┐
│         back_populates KEEPS BOTH SIDES IN SYNC                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  # If you set ONE side...                                       │
│  new_post = Post(title="X", author=alice)                       │
│                                                                 │
│  # ...the OTHER side updates automatically:                     │
│  alice.posts  →  [..., Post(title="X")]   ← new_post appeared! │
│                                                                 │
│                                                                 │
│  # Works the other way too:                                     │
│  another_post = Post(title="Y")                                 │
│  alice.posts.append(another_post)                               │
│                                                                 │
│  another_post.author  →  Author(name="Alice")   ← auto-set!    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.5 Many-to-Many: The Association Table

**In Week 5, Lecture 4, you learned: many-to-many requires a junction table.** Same rule in SQLAlchemy.

**A post can have many tags. A tag can be on many posts.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 MANY-TO-MANY STRUCTURE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   posts            post_tags              tags                  │
│  ┌────┐           ┌─────────┬────────┐   ┌────┐                │
│  │ id │──────────▶│ post_id │ tag_id │◀──│ id │                │
│  │    │           └─────────┴────────┘   │    │                │
│  └────┘            junction table        └────┘                │
│                    (no own id!)                                  │
│                                                                 │
│  Week 5 SQL:                                                    │
│  CREATE TABLE post_tags (                                       │
│      post_id INTEGER REFERENCES posts(id),                      │
│      tag_id INTEGER REFERENCES tags(id),                        │
│      PRIMARY KEY (post_id, tag_id)                              │
│  );                                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In SQLAlchemy, the association table is NOT a model class.** It's a plain `Table` object, because it has no data of its own — just two foreign keys:

```python
from sqlalchemy import Table, Column, ForeignKey

# Association table — NOT a class, just a Table
post_tags = Table(
    "post_tags",
    Base.metadata,
    Column("post_id", ForeignKey("posts.id"), primary_key=True),
    Column("tag_id", ForeignKey("tags.id"), primary_key=True),
)
# ▲
# This is the same junction table from Week 5,
# but expressed as a SQLAlchemy Table object.
# No Mapped[], no mapped_column() — it's not an ORM model.
# It's just plumbing that connects Post and Tag.
```

**Now define the Tag model and connect everything:**

```python
class Tag(Base):
    __tablename__ = "tags"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)

    # Many-to-many: a tag has many posts
    posts: Mapped[list["Post"]] = relationship(
        secondary=post_tags,         # ← "Use this junction table"
        back_populates="tags"
    )
```

**And add the other side to Post:**

```python
class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text)
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))

    # Many-to-one: a post has one author
    author: Mapped["Author"] = relationship(back_populates="posts")

    # Many-to-many: a post has many tags
    tags: Mapped[list["Tag"]] = relationship(
        secondary=post_tags,         # ← Same junction table
        back_populates="posts"
    )
```

```
┌─────────────────────────────────────────────────────────────────┐
│         THE secondary PARAMETER                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  relationship(secondary=post_tags)                              │
│               ▲                                                 │
│               │                                                 │
│  "To find related objects, go THROUGH this table."              │
│                                                                 │
│  When you access post.tags, SQLAlchemy:                         │
│  1. Looks in post_tags for rows matching this post's id         │
│  2. Gets the tag_ids from those rows                            │
│  3. Fetches the Tag objects with those ids                      │
│                                                                 │
│  Equivalent SQL (what you wrote in Week 5):                     │
│  SELECT tags.* FROM tags                                        │
│  JOIN post_tags ON tags.id = post_tags.tag_id                   │
│  WHERE post_tags.post_id = 1;                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.6 Working with Many-to-Many

**Adding and removing tags works like a Python list:**

```python
with Session(engine) as session:

    # Create tags
    python_tag = Tag(name="python")
    database_tag = Tag(name="databases")
    tutorial_tag = Tag(name="tutorial")

    # Create a post with tags
    post = Post(
        title="SQLAlchemy Relationships",
        content="How to connect models",
        author=alice
    )

    # Add tags — just append to the list!
    post.tags.append(python_tag)
    post.tags.append(database_tag)
    post.tags.append(tutorial_tag)

    session.add(post)
    session.commit()
```

**The SQL generated:**

```sql
INSERT INTO tags (name) VALUES ('python') RETURNING id       -- tag id=1
INSERT INTO tags (name) VALUES ('databases') RETURNING id    -- tag id=2
INSERT INTO tags (name) VALUES ('tutorial') RETURNING id     -- tag id=3

INSERT INTO posts (title, content, author_id) VALUES ('SQLAlchemy Relationships', ..., 1)
                                                                       -- post id=5

-- Junction table entries — SQLAlchemy handles this automatically:
INSERT INTO post_tags (post_id, tag_id) VALUES (5, 1)
INSERT INTO post_tags (post_id, tag_id) VALUES (5, 2)
INSERT INTO post_tags (post_id, tag_id) VALUES (5, 3)
```

**Navigating many-to-many in both directions:**

```python
with Session(engine) as session:

    # Post → Tags
    stmt = select(Post).where(Post.title == "SQLAlchemy Relationships")
    post = session.execute(stmt).scalars().first()

    print(f"Tags on '{post.title}':")
    for tag in post.tags:
        print(f"  #{tag.name}")
    # Output:
    #   #python
    #   #databases
    #   #tutorial


    # Tag → Posts (the other direction!)
    stmt = select(Tag).where(Tag.name == "python")
    python_tag = session.execute(stmt).scalars().first()

    print(f"Posts tagged 'python':")
    for post in python_tag.posts:
        print(f"  {post.title}")
    # Output:
    #   SQLAlchemy Relationships
    #   (any other posts with the python tag)
```

**Removing a tag:**

```python
with Session(engine) as session:

    stmt = select(Post).where(Post.title == "SQLAlchemy Relationships")
    post = session.execute(stmt).scalars().first()

    # Find the tag to remove
    tag_to_remove = next(t for t in post.tags if t.name == "tutorial")

    # Remove it — just like a Python list
    post.tags.remove(tag_to_remove)

    session.commit()
    # SQLAlchemy deletes the row from post_tags automatically.
    # The Tag itself is NOT deleted — just the association.
```

```sql
-- SQLAlchemy generates:
DELETE FROM post_tags WHERE post_id = 5 AND tag_id = 3
-- Only the junction row is removed. The tag "tutorial" still exists.
```

---

## Complete Model Definitions (Reference)

**Here's the full picture — all three models connected. We'll use these for the rest of the lecture:**

```python
from sqlalchemy import (
    create_engine, ForeignKey, String, Text,
    Table, Column, func
)
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, relationship
)
from datetime import datetime
from typing import Optional


class Base(DeclarativeBase):
    pass


# Association table for many-to-many (Post ↔ Tag)
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

    # One-to-many: one author has many posts
    posts: Mapped[list["Post"]] = relationship(back_populates="author")

    def __repr__(self) -> str:
        return f"Author(id={self.id}, name='{self.name}')"


class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text)
    published_at: Mapped[datetime] = mapped_column(default=func.now())
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))

    # Many-to-one: one post has one author
    author: Mapped["Author"] = relationship(back_populates="posts")

    # Many-to-many: a post has many tags
    tags: Mapped[list["Tag"]] = relationship(
        secondary=post_tags, back_populates="posts"
    )

    def __repr__(self) -> str:
        return f"Post(id={self.id}, title='{self.title}')"


class Tag(Base):
    __tablename__ = "tags"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)

    # Many-to-many: a tag is on many posts
    posts: Mapped[list["Post"]] = relationship(
        secondary=post_tags, back_populates="tags"
    )

    def __repr__(self) -> str:
        return f"Tag(id={self.id}, name='{self.name}')"
```

```
┌─────────────────────────────────────────────────────────────────┐
│              THE COMPLETE RELATIONSHIP MAP                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    Author                                       │
│                 ┌──────────┐                                    │
│                 │ id       │                                    │
│                 │ name     │                                    │
│                 │ email    │                                    │
│                 │          │                                    │
│                 │ posts ───────┐  (one-to-many)                 │
│                 └──────────┘   │                                │
│                                │                                │
│                                ▼                                │
│    Tag                      Post                                │
│  ┌──────────┐            ┌──────────────┐                       │
│  │ id       │            │ id           │                       │
│  │ name     │            │ title        │                       │
│  │          │◀──────────▶│ content      │                       │
│  │ posts    │ (many-to-  │ published_at │                       │
│  └──────────┘   many)    │ author_id FK │                       │
│                          │              │                       │
│                          │ author       │ (many-to-one)         │
│                          │ tags         │ (many-to-many)        │
│                          └──────────────┘                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: QUERYING

## 3.1 select() — The Foundation (SQLAlchemy 2.0)

**In Week 5, you wrote raw SQL queries. In Week 6 Lecture 1, you saw `select()` briefly. Now let's master it.**

```python
from sqlalchemy import select

# The basic pattern — you'll write this hundreds of times:
stmt = select(Author)                            # Build the statement
result = session.execute(stmt)                   # Execute it
authors = result.scalars().all()                 # Extract the results
```

**What does each piece do?**

```
┌─────────────────────────────────────────────────────────────────┐
│             THE QUERY PIPELINE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: BUILD THE STATEMENT                                    │
│  ─────────────────────────                                      │
│  stmt = select(Author).where(...).order_by(...)                 │
│  ▲                                                              │
│  This is just a DESCRIPTION of what you want.                   │
│  No database call yet. Like writing the SQL on paper.           │
│                                                                 │
│                                                                 │
│  Step 2: EXECUTE THE STATEMENT                                  │
│  ──────────────────────────────                                 │
│  result = session.execute(stmt)                                 │
│  ▲                                                              │
│  NOW the query hits the database. Returns a Result object.      │
│  Like handing your SQL to the database and getting a response.  │
│                                                                 │
│                                                                 │
│  Step 3: EXTRACT WHAT YOU NEED                                  │
│  ──────────────────────────────                                 │
│  authors = result.scalars().all()                               │
│                   ▲          ▲                                   │
│                   │          └─ .all() = collect into a list     │
│                   │             .first() = get first or None     │
│                   │             .one() = get exactly one or raise│
│                   │                                             │
│                   └─ .scalars() = "I want the ORM objects,      │
│                       not Row tuples"                            │
│                                                                 │
│                      Without scalars():                          │
│                        [(Author(...),), (Author(...),)]         │
│                      With scalars():                             │
│                        [Author(...), Author(...)]               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The SQL it generates:**

```python
stmt = select(Author)
```

```sql
-- Equivalent SQL (what you wrote in Week 5):
SELECT authors.id, authors.name, authors.email
FROM authors
```

**Selecting specific columns (less common with ORM, but useful):**

```python
# Select specific columns — returns tuples, not objects
stmt = select(Author.name, Author.email)
result = session.execute(stmt)
rows = result.all()  # [("Alice", "alice@..."), ("Bob", "bob@...")]
#                       ▲
#                       No .scalars() here — you're getting tuples,
#                       not Author objects
```

---

## 3.2 where() — Filtering Results

**`where()` is your `WHERE` clause from SQL, built in Python.**

```python
# Find a specific author
stmt = select(Author).where(Author.name == "Alice")
alice = session.execute(stmt).scalars().first()
```

```sql
SELECT * FROM authors WHERE authors.name = 'Alice'
```

**SQLAlchemy overloads Python's comparison operators:**

```python
# These Python expressions become SQL conditions:

Author.name == "Alice"              # name = 'Alice'
Author.id != 1                      # id != 1
Author.id > 5                       # id > 5
Author.id >= 5                      # id >= 5
Author.id < 10                      # id < 10
Post.published_at <= some_date      # published_at <= '2025-01-01'
```

**String-specific filters (you used LIKE in Week 5):**

```python
# LIKE patterns — same concept as Week 5 SQL
Author.name.like("A%")             # name LIKE 'A%'         (starts with A)
Author.name.ilike("%alice%")       # name ILIKE '%alice%'   (case-insensitive)
Author.name.contains("li")        # name LIKE '%li%'       (contains "li")
Author.name.startswith("A")       # name LIKE 'A%'
Author.name.endswith("ce")        # name LIKE '%ce'
```

**NULL checks:**

```python
# IS NULL / IS NOT NULL
Post.content.is_(None)              # content IS NULL
Post.content.is_not(None)           # content IS NOT NULL
#           ▲
#           Note: is_() and is_not(), not == None!
#           == None works but generates a warning.
```

**IN clause:**

```python
# IN — you used this in Week 5 SQL too
Author.id.in_([1, 2, 3])           # id IN (1, 2, 3)
Author.name.in_(["Alice", "Bob"])   # name IN ('Alice', 'Bob')

# NOT IN
Author.id.not_in([1, 2])           # id NOT IN (1, 2)
```

---

## 3.3 Combining Filters (and_, or_, in_)

**Multiple conditions — multiple ways to express them:**

```python
from sqlalchemy import and_, or_

# Method 1: Multiple arguments to where() — implicitly AND
stmt = select(Post).where(
    Post.author_id == 1,
    Post.published_at >= some_date
)
# WHERE author_id = 1 AND published_at >= '2025-01-01'


# Method 2: Chaining .where() calls — also AND
stmt = (
    select(Post)
    .where(Post.author_id == 1)
    .where(Post.published_at >= some_date)
)
# Same SQL as above


# Method 3: Explicit and_() — when you need to be clear
stmt = select(Post).where(
    and_(
        Post.author_id == 1,
        Post.published_at >= some_date
    )
)
# Same SQL as above
```

**OR conditions — requires explicit or_():**

```python
# Posts by Alice (id=1) OR Bob (id=2)
stmt = select(Post).where(
    or_(
        Post.author_id == 1,
        Post.author_id == 2
    )
)
# WHERE author_id = 1 OR author_id = 2

# Better way for this specific case:
stmt = select(Post).where(Post.author_id.in_([1, 2]))
# WHERE author_id IN (1, 2)
```

**Complex combinations — nest and_() and or_():**

```python
# Posts that are:
#   (by Alice AND published after Jan 1) OR (tagged as "tutorial")
#
# In SQL:
# WHERE (author_id = 1 AND published_at >= '2025-01-01')
#    OR title LIKE '%tutorial%'

stmt = select(Post).where(
    or_(
        and_(
            Post.author_id == 1,
            Post.published_at >= datetime(2025, 1, 1)
        ),
        Post.title.contains("tutorial")
    )
)
```

```
┌─────────────────────────────────────────────────────────────────┐
│                FILTER COMBINATIONS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Multiple .where() or comma-separated = AND                     │
│  ───────────────────────────────────────                        │
│  .where(A, B)         →  WHERE A AND B                          │
│  .where(A).where(B)   →  WHERE A AND B                          │
│  .where(and_(A, B))   →  WHERE A AND B                          │
│                                                                 │
│  or_() required for OR:                                         │
│  ──────────────────────                                         │
│  .where(or_(A, B))    →  WHERE A OR B                           │
│                                                                 │
│  Nesting for complex logic:                                     │
│  ──────────────────────────                                     │
│  .where(                                                        │
│      or_(                                                       │
│          and_(A, B),     →  WHERE (A AND B) OR C                │
│          C                                                      │
│      )                                                          │
│  )                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Ordering and Limiting (order_by, limit)

**Sorting — you used ORDER BY in Week 5:**

```python
# Ascending (default)
stmt = select(Post).order_by(Post.published_at)
# ORDER BY published_at ASC

# Descending — newest first
stmt = select(Post).order_by(Post.published_at.desc())
# ORDER BY published_at DESC

# Multiple sort keys
stmt = select(Post).order_by(Post.author_id, Post.published_at.desc())
# ORDER BY author_id ASC, published_at DESC
```

**Limiting results:**

```python
# Get first 10 posts
stmt = select(Post).order_by(Post.published_at.desc()).limit(10)
# SELECT * FROM posts ORDER BY published_at DESC LIMIT 10

# Skip 20, get next 10 (we'll revisit this in pagination)
stmt = select(Post).order_by(Post.id).offset(20).limit(10)
# SELECT * FROM posts ORDER BY id LIMIT 10 OFFSET 20
```

**Chaining it all together:**

```python
# Real-world query: recent posts by Alice, first 5
stmt = (
    select(Post)
    .where(Post.author_id == 1)
    .order_by(Post.published_at.desc())
    .limit(5)
)

posts = session.execute(stmt).scalars().all()
```

```sql
-- The SQL generated:
SELECT posts.id, posts.title, posts.content, posts.published_at, posts.author_id
FROM posts
WHERE posts.author_id = 1
ORDER BY posts.published_at DESC
LIMIT 5
```

> "Look familiar? It's the exact same SQL you wrote by hand in Week 5. SQLAlchemy just lets you write it in Python, with type checking, IDE autocomplete, and composability."

---

## 3.5 Querying Across Relationships (join)

**This is where relationships and queries meet. You can filter by related model attributes.**

**The question: "Find all posts written by Alice."**

```python
# Without join — if you already know Alice's id:
stmt = select(Post).where(Post.author_id == 1)

# With join — if you only know her NAME:
stmt = (
    select(Post)
    .join(Post.author)               # JOIN authors ON posts.author_id = authors.id
    .where(Author.name == "Alice")   # WHERE authors.name = 'Alice'
)

posts = session.execute(stmt).scalars().all()
```

```sql
-- SQL generated:
SELECT posts.id, posts.title, posts.content, posts.published_at, posts.author_id
FROM posts
JOIN authors ON posts.author_id = authors.id
WHERE authors.name = 'Alice'
```

> "Compare this to the JOIN you wrote in Week 5, Lecture 2. Same concept — `.join(Post.author)` means *'connect posts to authors through the relationship we defined.'*"

**More complex: find posts that have a specific tag.**

```python
# Posts tagged with "python"
stmt = (
    select(Post)
    .join(Post.tags)                 # Goes through the post_tags junction table
    .where(Tag.name == "python")
)

python_posts = session.execute(stmt).scalars().all()
```

```sql
-- SQL generated — the many-to-many JOIN:
SELECT posts.*
FROM posts
JOIN post_tags ON posts.id = post_tags.post_id
JOIN tags ON tags.id = post_tags.tag_id
WHERE tags.name = 'python'
```

```
┌─────────────────────────────────────────────────────────────────┐
│         .join() USES YOUR RELATIONSHIP DEFINITIONS              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  .join(Post.author)                                             │
│         ▲                                                       │
│         └─ SQLAlchemy reads the relationship definition,        │
│            knows the foreign key, and generates the             │
│            correct JOIN clause. You never specify               │
│            ON posts.author_id = authors.id manually.            │
│                                                                 │
│                                                                 │
│  .join(Post.tags)                                               │
│         ▲                                                       │
│         └─ SQLAlchemy sees secondary=post_tags,                 │
│            generates a TWO-TABLE JOIN through the               │
│            junction table automatically.                        │
│                                                                 │
│  This is why we defined relationships in Part 2.                │
│  They power your queries.                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Combining joins with other filters:**

```python
# Alice's posts tagged "python", newest first, top 5
stmt = (
    select(Post)
    .join(Post.author)
    .join(Post.tags)
    .where(
        Author.name == "Alice",
        Tag.name == "python"
    )
    .order_by(Post.published_at.desc())
    .limit(5)
)
```

```sql
SELECT posts.*
FROM posts
JOIN authors ON posts.author_id = authors.id
JOIN post_tags ON posts.id = post_tags.post_id
JOIN tags ON tags.id = post_tags.tag_id
WHERE authors.name = 'Alice' AND tags.name = 'python'
ORDER BY posts.published_at DESC
LIMIT 5
```

---

# PART 4: THE N+1 PROBLEM

## 4.1 The Phone Call Problem (Demonstration)

**This is the single most important concept in this lecture. It will haunt every ORM you ever use — SQLAlchemy, Django ORM, TypeORM, ActiveRecord. Get this right.**

**The setup: we want to list all authors and their post titles.**

```python
# Seed data in the database:
# Alice  → 3 posts
# Bob    → 2 posts
# Charlie → 4 posts
# Diana  → 1 post
# Eve    → 3 posts
# ... (100 authors total, each with some posts)
```

**The naive approach — looks innocent, performs terribly:**

```python
# engine with echo=True so we can SEE every SQL query
engine = create_engine("postgresql://user:pass@localhost/blog", echo=True)

with Session(engine) as session:

    # Step 1: Get all authors
    stmt = select(Author)
    authors = session.execute(stmt).scalars().all()

    # Step 2: Print each author's posts
    for author in authors:
        print(f"\n{author.name}'s posts:")
        for post in author.posts:    # ← THIS LINE IS THE PROBLEM
            print(f"  - {post.title}")
```

**Watch the console. Count the queries:**

```sql
-- Query 1: Get all authors
SELECT authors.id, authors.name, authors.email FROM authors

-- Query 2: When we access alice.posts...
SELECT posts.id, posts.title, posts.content, posts.author_id
FROM posts WHERE posts.author_id = 1

-- Query 3: When we access bob.posts...
SELECT posts.id, posts.title, posts.content, posts.author_id
FROM posts WHERE posts.author_id = 2

-- Query 4: When we access charlie.posts...
SELECT posts.id, posts.title, posts.content, posts.author_id
FROM posts WHERE posts.author_id = 3

-- Query 5...
-- Query 6...
-- ...
-- Query 101: When we access the 100th author's posts...
SELECT posts.id, posts.title, posts.content, posts.author_id
FROM posts WHERE posts.author_id = 100
```

**Now ask the class:**

> "How many queries did we just make?"

Answer: **101 queries.** 1 to get all authors + 100 to get each author's posts. This is the **N+1 problem** — 1 initial query, then N follow-up queries.

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE N+1 PROBLEM                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THE PHONE CALL ANALOGY                                         │
│  ──────────────────────                                         │
│                                                                 │
│  Imagine every SQL query is a PHONE CALL to the database.       │
│  Each call has overhead: connection, network latency,           │
│  the database has to parse, plan, and execute.                  │
│                                                                 │
│  N+1 approach:                                                  │
│  ─────────────                                                  │
│  📞 Call 1:   "Give me all 100 authors."                        │
│  📞 Call 2:   "What posts did author #1 write?"                 │
│  📞 Call 3:   "What posts did author #2 write?"                 │
│  📞 Call 4:   "What posts did author #3 write?"                 │
│  📞 ...                                                         │
│  📞 Call 101: "What posts did author #100 write?"               │
│                                                                 │
│  101 phone calls. Each one waits for a response.                │
│  Each one has ~1-5ms of network latency.                        │
│                                                                 │
│  At 2ms per query: 101 × 2ms = 202ms                            │
│  At 5ms per query: 101 × 5ms = 505ms ← HALF A SECOND           │
│  With 1000 authors: 1001 × 5ms = 5 SECONDS                     │
│                                                                 │
│  For a single API endpoint!                                     │
│                                                                 │
│                                                                 │
│  The efficient way (coming next):                               │
│  ─────────────────────────────────                              │
│  📞 Call 1: "Give me all authors AND their posts together."     │
│                                                                 │
│  1 phone call. Done.                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Lazy Loading (The Default Trap)

**What just happened? SQLAlchemy's default loading strategy is "lazy loading."**

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAZY LOADING                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "Don't load related objects until someone ACCESSES them."      │
│                                                                 │
│  author = session.execute(select(Author)...).scalars().first()  │
│  # At this point, author.posts is NOT loaded.                   │
│  # SQLAlchemy knows the relationship EXISTS,                    │
│  # but hasn't fetched the data yet.                             │
│                                                                 │
│  print(author.posts)   ← ACCESS triggers a query                │
│  # NOW SQLAlchemy fires:                                        │
│  # SELECT * FROM posts WHERE author_id = 1                      │
│                                                                 │
│                                                                 │
│  WHY IS THIS THE DEFAULT?                                       │
│  ─────────────────────────                                      │
│  It's not stupid — it avoids loading data you DON'T need.       │
│  If you only need the author's name, why load 500 posts?        │
│                                                                 │
│  WHEN DOES IT BECOME A PROBLEM?                                 │
│  ──────────────────────────────                                 │
│  When you access the relationship IN A LOOP.                    │
│  Each iteration triggers a new query.                           │
│  That's N+1.                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The danger is that it's invisible.** The code looks perfectly normal:

```python
for author in authors:
    print(author.posts)  # This innocent line fires N queries
```

There's no error, no warning. Your code is just slow and you don't know why — unless you turn on `echo=True` or use logging.

**And there's another trap — the DetachedInstanceError:**

```python
with Session(engine) as session:
    stmt = select(Author).where(Author.name == "Alice")
    alice = session.execute(stmt).scalars().first()

# Session is closed now!

print(alice.name)   # ✅ Works — name was loaded with the author
print(alice.posts)  # ❌ DetachedInstanceError!
#                       SQLAlchemy tries to lazy-load posts,
#                       but the session is gone. No database connection.
#                       Can't make the phone call.
```

```
┌─────────────────────────────────────────────────────────────────┐
│              DetachedInstanceError                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Lazy loading needs an ACTIVE session to fire queries.          │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │   Session     │    │   Author     │    │  Database    │       │
│  │   (open)     │◀──▶│   .posts     │──▶ │  (query)     │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│         ✅ Works: session can relay the query                    │
│                                                                 │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │   Session     │    │   Author     │    │  Database    │       │
│  │   (CLOSED)   │ ✗  │   .posts     │──▶ │  ???         │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│         ❌ DetachedInstanceError: no session to query through    │
│                                                                 │
│  Solution: Load the data BEFORE the session closes.             │
│  That's what eager loading is for.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 joinedload() — One Big Query

**The first eager loading strategy: load everything in a single JOIN query.**

```python
from sqlalchemy.orm import joinedload

with Session(engine) as session:

    # Tell SQLAlchemy: "When you fetch authors, also fetch their posts"
    stmt = select(Author).options(joinedload(Author.posts))
    #                      ▲              ▲
    #                      │              └─ "Eager-load this relationship"
    #                      └─ .options() = loading instructions

    authors = session.execute(stmt).scalars().unique().all()
    #                                        ▲
    #                                        └─ .unique() is needed with
    #                                           joinedload to deduplicate
    #                                           (one author per row, not
    #                                            one per post)

# ALL data is loaded. Session can close safely.
for author in authors:
    print(f"{author.name}'s posts:")
    for post in author.posts:      # ← NO extra queries! Already loaded.
        print(f"  - {post.title}")
```

**The SQL — one single query with a JOIN:**

```sql
-- ONE query instead of 101:
SELECT authors.id, authors.name, authors.email,
       posts.id AS posts_id, posts.title, posts.content,
       posts.published_at, posts.author_id
FROM authors
LEFT OUTER JOIN posts ON authors.id = posts.author_id
```

```
┌─────────────────────────────────────────────────────────────────┐
│              joinedload — HOW IT WORKS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  📞 ONE phone call:                                             │
│  "Give me all authors LEFT JOINed with their posts."            │
│                                                                 │
│  The database returns:                                          │
│  ┌─────────┬───────────┬──────────────────────┐                 │
│  │ author  │ author    │ post                  │                │
│  │ id      │ name      │ title                 │                │
│  ├─────────┼───────────┼──────────────────────┤                 │
│  │ 1       │ Alice     │ Hello World           │                │
│  │ 1       │ Alice     │ Async Deep Dive       │  ← Alice      │
│  │ 1       │ Alice     │ Type Hints Guide      │    repeated!   │
│  │ 2       │ Bob       │ REST API Design       │                │
│  │ 2       │ Bob       │ Testing Strategies    │  ← Bob         │
│  │ 3       │ Charlie   │ NULL                  │    repeated!   │
│  └─────────┴───────────┴──────────────────────┘                 │
│                          ▲                                      │
│                          Charlie has no posts (LEFT JOIN)        │
│                                                                 │
│  SQLAlchemy deduplicates this into:                             │
│  Author("Alice")  →  [Post("Hello"), Post("Async"), Post(...)] │
│  Author("Bob")    →  [Post("REST"), Post("Testing")]           │
│  Author("Charlie") →  []                                       │
│                                                                 │
│  That's why you need .unique()!                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When is joinedload good?**

```
┌─────────────────────────────────────────────────────────────────┐
│              joinedload — TRADEOFFS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ GOOD WHEN:                                                   │
│  • Related objects are small (few columns)                      │
│  • The "many" side is not too many (1-20 per parent)            │
│  • You want ONE database round-trip                             │
│                                                                 │
│  ❌ BAD WHEN:                                                    │
│  • The "many" side is HUGE (100+ posts per author)              │
│  • Data gets duplicated across rows (wasteful bandwidth)        │
│  • Many-to-many with large associations                         │
│  • Multiple joinedloads on the same query (Cartesian product!)  │
│                                                                 │
│  The duplication problem:                                       │
│  If Alice has 100 posts, Alice's name/email is repeated         │
│  100 times in the result. That's wasted network bandwidth.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 selectinload() — Two Smart Queries

**The second eager loading strategy: load in two separate, efficient queries.**

```python
from sqlalchemy.orm import selectinload

with Session(engine) as session:

    stmt = select(Author).options(selectinload(Author.posts))
    #                                ▲
    #                                selectinload instead of joinedload

    authors = session.execute(stmt).scalars().all()
    #                                        ▲
    #                                No .unique() needed with selectinload!

for author in authors:
    print(f"{author.name}'s posts:")
    for post in author.posts:      # ← NO extra queries!
        print(f"  - {post.title}")
```

**The SQL — two queries, no duplication:**

```sql
-- Query 1: Get all authors
SELECT authors.id, authors.name, authors.email
FROM authors

-- Query 2: Get ALL posts for those authors in ONE query, using IN
SELECT posts.id, posts.title, posts.content, posts.published_at, posts.author_id
FROM posts
WHERE posts.author_id IN (1, 2, 3, 4, 5, ... 100)
```

```
┌─────────────────────────────────────────────────────────────────┐
│              selectinload — HOW IT WORKS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  📞 Phone call 1:                                               │
│  "Give me all authors."                                         │
│  → Gets 100 authors. Collects their IDs: [1, 2, 3, ..., 100]   │
│                                                                 │
│  📞 Phone call 2:                                               │
│  "Give me all posts WHERE author_id IN (1, 2, ..., 100)."      │
│  → Gets ALL posts for ALL authors in one shot.                  │
│                                                                 │
│  SQLAlchemy matches them up in memory:                          │
│  Author(id=1).posts = [posts where author_id=1]                │
│  Author(id=2).posts = [posts where author_id=2]                │
│  ...                                                            │
│                                                                 │
│  TOTAL: 2 queries. No duplication. No N+1.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why no `.unique()` needed?**

> "Unlike joinedload, selectinload doesn't JOIN — it uses separate queries. Each author appears only once in the first query. No duplication to deduplicate."

**When is selectinload good?**

```
┌─────────────────────────────────────────────────────────────────┐
│              selectinload — TRADEOFFS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ GOOD WHEN:                                                   │
│  • The "many" side is large (many posts per author)             │
│  • You want to avoid data duplication                           │
│  • Loading multiple relationships at once                       │
│  • Many-to-many relationships                                   │
│                                                                 │
│  ❌ BAD WHEN:                                                    │
│  • The IN clause has too many IDs (thousands — rare)            │
│  • You need exactly one round-trip (joinedload wins)            │
│                                                                 │
│  Generally, selectinload is the SAFER default choice.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 Choosing Your Loading Strategy

**The complete comparison:**

```
┌─────────────────────────────────────────────────────────────────┐
│         LOADING STRATEGY COMPARISON                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│               Lazy           joinedload       selectinload      │
│               (default)                                         │
│  ─────────────────────────────────────────────────────────────  │
│  Queries:     1 + N          1                 2                │
│                                                                 │
│  SQL:         SELECT each    LEFT JOIN         SELECT +         │
│               one-by-one     in one query      WHERE IN         │
│                                                                 │
│  Duplicates:  No             Yes (parent       No               │
│                              rows repeated)                     │
│                                                                 │
│  .unique():   No             YES (required)    No               │
│                                                                 │
│  Best for:    Single object  Small related     Large related    │
│               access, when   sets, need one    sets, multiple   │
│               you might not  round-trip        relationships    │
│               need related                                      │
│               data at all                                       │
│                                                                 │
│  Danger:      N+1 in loops!  Cartesian         Very large IN    │
│                              product with      clauses (rare)   │
│                              multiple joins                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The decision framework:**

```
┌─────────────────────────────────────────────────────────────────┐
│           WHICH LOADING STRATEGY?                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│           Will you access related objects?                       │
│                    │                                            │
│              YES   │   NO                                       │
│              │     │   │                                        │
│              │     │   └─▶ Lazy loading is fine.                │
│              │     │       Don't load what you don't use.       │
│              ▼     │                                            │
│     Is it in a loop                                             │
│     (multiple parents)?                                         │
│              │                                                  │
│        YES   │   NO                                             │
│        │     │   │                                              │
│        │     │   └─▶ Lazy loading is fine.                     │
│        │     │       One query for one object is OK.            │
│        ▼     │                                                  │
│     How large is the                                            │
│     "many" side?                                                │
│        │                                                        │
│   Small │   Large                                               │
│   (≤20) │   (>20)                                               │
│     │   │     │                                                 │
│     ▼   │     ▼                                                 │
│  joinedload  selectinload                                       │
│  (1 query)   (2 queries)                                        │
│                                                                 │
│  When in doubt? Use selectinload. It's almost always safe.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Loading multiple relationships at once:**

```python
# Load author's posts AND each post's tags in one go
stmt = (
    select(Author)
    .options(
        selectinload(Author.posts).selectinload(Post.tags)
        #                         ▲
        #                         Chain them! Load posts,
        #                         then for each post, load tags.
    )
)

# This generates 3 queries total:
# 1. SELECT * FROM authors
# 2. SELECT * FROM posts WHERE author_id IN (...)
# 3. SELECT * FROM tags JOIN post_tags WHERE post_id IN (...)
#
# Compare to lazy loading: 1 + N + (N × M) queries!
```

---

# PART 5: PAGINATION

## 5.1 Offset-Based Pagination

**Connection to Week 4, Lecture 2: you learned about pagination in API design. Now let's implement it at the database layer.**

**The simplest approach — OFFSET and LIMIT:**

```python
def get_posts_page(session: Session, page: int, page_size: int = 10) -> list[Post]:
    """Get a specific page of posts."""
    offset = (page - 1) * page_size
    #         ▲
    #         page 1 → offset 0   (skip nothing)
    #         page 2 → offset 10  (skip first 10)
    #         page 3 → offset 20  (skip first 20)

    stmt = (
        select(Post)
        .order_by(Post.id)      # Consistent ordering is REQUIRED
        .offset(offset)
        .limit(page_size)
    )

    return session.execute(stmt).scalars().all()


# Usage:
page1 = get_posts_page(session, page=1)   # Posts 1-10
page2 = get_posts_page(session, page=2)   # Posts 11-20
page3 = get_posts_page(session, page=3)   # Posts 21-30
```

```sql
-- Page 1:
SELECT * FROM posts ORDER BY id LIMIT 10 OFFSET 0

-- Page 2:
SELECT * FROM posts ORDER BY id LIMIT 10 OFFSET 10

-- Page 3:
SELECT * FROM posts ORDER BY id LIMIT 10 OFFSET 20
```

**Simple, intuitive, works for small datasets.** But there's a problem.

---

## 5.2 Why Offset Breaks at Scale

**Run it. Watch what happens as page numbers grow:**

```
┌─────────────────────────────────────────────────────────────────┐
│            THE OFFSET PROBLEM                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your table has 1,000,000 posts.                                │
│                                                                 │
│  Page 1 (OFFSET 0):                                             │
│  ─────────────────                                              │
│  Database reads 10 rows. Returns them.                          │
│  Fast. ✅                                                        │
│                                                                 │
│  Page 100 (OFFSET 990):                                         │
│  ──────────────────────                                         │
│  Database reads 1,000 rows. Throws away 990. Returns 10.       │
│  Getting slower. 🤔                                              │
│                                                                 │
│  Page 10,000 (OFFSET 99,990):                                   │
│  ─────────────────────────────                                  │
│  Database reads 100,000 rows. Throws away 99,990. Returns 10.  │
│  SLOW. ❌                                                        │
│                                                                 │
│  Page 100,000 (OFFSET 999,990):                                 │
│  ──────────────────────────────                                 │
│  Database reads 1,000,000 rows. Throws away 999,990. Returns 10│
│  YOUR SERVER IS DYING. 💀                                        │
│                                                                 │
│                                                                 │
│  The problem: the database MUST COUNT through all               │
│  skipped rows. OFFSET doesn't magically jump ahead.             │
│  It scans from the start every time.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
  Response time vs. Page number:

  Time │
  (ms) │
  500  │                                          ╱
       │                                        ╱
  400  │                                      ╱
       │                                   ╱
  300  │                                ╱
       │                            ╱
  200  │                        ╱
       │                   ╱
  100  │             ╱╱
       │        ╱╱
    0  │───╱╱
       └─────────────────────────────────────────
         1     100    1K     10K    50K   100K
                       Page number

  Linear degradation. The deeper you go, the slower it gets.
```

**There's also a consistency problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│            OFFSET CONSISTENCY BUG                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You request Page 1 (OFFSET 0, LIMIT 10): get posts 1-10       │
│                                                                 │
│  While you're reading, someone INSERTS a new post at position 5.│
│                                                                 │
│  You request Page 2 (OFFSET 10, LIMIT 10):                      │
│  All rows shifted. Post 10 is now at position 11.               │
│  You see post 10 AGAIN. Duplicate!                              │
│                                                                 │
│  Or if someone DELETES a post: you SKIP one.                    │
│                                                                 │
│  OFFSET pagination is not stable in a changing dataset.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Keyset-Based (Cursor) Pagination

**Instead of "skip N rows," use "give me rows AFTER this one."**

```python
from typing import Optional

def get_posts_after(
    session: Session,
    cursor: Optional[int] = None,   # The last post ID we saw
    page_size: int = 10,
) -> list[Post]:
    """Get the next page of posts after the given cursor."""

    stmt = select(Post).order_by(Post.id).limit(page_size)

    if cursor is not None:
        stmt = stmt.where(Post.id > cursor)
        #                        ▲
        #                        "Give me posts with id GREATER THAN
        #                         the last one I saw."
        #                        The database uses the INDEX to jump
        #                        directly there. No scanning!

    return session.execute(stmt).scalars().all()


# Usage:
page1 = get_posts_after(session, cursor=None)     # First page
# page1[-1].id = 10

page2 = get_posts_after(session, cursor=10)        # After id 10
# page2[-1].id = 20

page3 = get_posts_after(session, cursor=20)        # After id 20
```

```sql
-- Page 1:
SELECT * FROM posts ORDER BY id LIMIT 10
-- Database returns posts 1-10 instantly. ✅

-- Page 2 (cursor = 10):
SELECT * FROM posts WHERE id > 10 ORDER BY id LIMIT 10
-- Database jumps to id=11 using the index. Returns 11-20. ✅

-- Page 10,000 (cursor = 99990):
SELECT * FROM posts WHERE id > 99990 ORDER BY id LIMIT 10
-- Database jumps to id=99991 using the index. Returns 10 rows. STILL FAST. ✅
```

**Why is this fast?**

```
┌─────────────────────────────────────────────────────────────────┐
│          WHY KEYSET PAGINATION IS FAST                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OFFSET (skip and count):                                       │
│                                                                 │
│  Start ──────────────────────────────────────▶ Target           │
│  │ 1 │ 2 │ 3 │ ... │ 99,989 │ 99,990 │ 99,991│ ... │          │
│  └─skip──skip──skip──skip──skip──skip──┘ ▲                      │
│                                          │                      │
│  Must touch every row to count to offset. │                      │
│                                                                 │
│                                                                 │
│  KEYSET (jump to position):                                     │
│                                                                 │
│  WHERE id > 99990                                               │
│  ─────────────────────────────────────────▶                     │
│                                     │ 99,991 │ 99,992 │ ...    │
│                                     ▲                           │
│  Index lookup. Jump directly.       │                           │
│  No scanning. O(log n) instead of O(n).                         │
│                                                                 │
│  ┌──────────────────────────────────────────┐                   │
│  │ EXPLAIN ANALYZE tip (Week 5, Lecture 3): │                   │
│  │ Run EXPLAIN on both queries.             │                   │
│  │ OFFSET → Seq Scan, high row count        │                   │
│  │ WHERE > → Index Scan, low row count      │                   │
│  └──────────────────────────────────────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Multi-column cursors (when id isn't enough):**

```python
# Sorting by published_at (non-unique), then id (tiebreaker)
def get_posts_after(
    session: Session,
    cursor_date: Optional[datetime] = None,
    cursor_id: Optional[int] = None,
    page_size: int = 10,
) -> list[Post]:

    stmt = (
        select(Post)
        .order_by(Post.published_at.desc(), Post.id.desc())
        .limit(page_size)
    )

    if cursor_date is not None and cursor_id is not None:
        # "Give me posts older than cursor_date,
        #  OR same date but smaller id"
        stmt = stmt.where(
            or_(
                Post.published_at < cursor_date,
                and_(
                    Post.published_at == cursor_date,
                    Post.id < cursor_id
                )
            )
        )

    return session.execute(stmt).scalars().all()
```

> "This is more complex than simple offset, but it's the pattern you'll use in production. The cursor encodes your position, not a page number."

---

## 5.4 The Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│         OFFSET vs KEYSET — WHEN TO USE WHICH                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                 OFFSET PAGINATION                                │
│                 ─────────────────                                │
│  ✅ Use when:                                                    │
│  • Small datasets (< 10,000 rows)                               │
│  • Users need "jump to page 50" functionality                   │
│  • Admin dashboards, internal tools                             │
│  • Simplicity matters more than performance                     │
│                                                                 │
│  ❌ Avoid when:                                                  │
│  • Large datasets (> 100,000 rows)                              │
│  • Data changes frequently (inserts/deletes)                    │
│  • API endpoints consumed by mobile apps (battery/bandwidth)    │
│                                                                 │
│                                                                 │
│                 KEYSET PAGINATION                                │
│                 ─────────────────                                │
│  ✅ Use when:                                                    │
│  • Large datasets                                               │
│  • Infinite scroll / "load more" UI                             │
│  • Real-time feeds (new items appearing)                        │
│  • API endpoints needing consistent performance                 │
│                                                                 │
│  ❌ Limitations:                                                 │
│  • Can't jump to arbitrary page (only "next" / "previous")     │
│  • Requires a unique, orderable column (or combination)         │
│  • More complex cursor encoding                                 │
│                                                                 │
│                                                                 │
│  Rule of thumb:                                                 │
│  ─────────────                                                  │
│  Start with offset. Switch to keyset when it gets slow.         │
│  Most student projects: offset is perfectly fine.               │
│  Most production APIs: keyset is worth the complexity.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│           RELATIONSHIPS & QUERYING — QUICK REFERENCE            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DEFINE ONE-TO-MANY:                                            │
│    class Parent(Base):                                          │
│        children: Mapped[list["Child"]] = relationship(          │
│            back_populates="parent")                             │
│    class Child(Base):                                           │
│        parent_id: Mapped[int] = mapped_column(                  │
│            ForeignKey("parents.id"))                            │
│        parent: Mapped["Parent"] = relationship(                 │
│            back_populates="children")                           │
│                                                                 │
│  DEFINE MANY-TO-MANY:                                           │
│    assoc = Table("assoc", Base.metadata,                        │
│        Column("a_id", ForeignKey("a.id"), primary_key=True),    │
│        Column("b_id", ForeignKey("b.id"), primary_key=True))    │
│    class A(Base):                                               │
│        bs: Mapped[list["B"]] = relationship(                    │
│            secondary=assoc, back_populates="as_")               │
│                                                                 │
│  BASIC QUERY:                                                   │
│    stmt = select(Model).where(Model.col == val)                 │
│    result = session.execute(stmt).scalars().all()               │
│                                                                 │
│  FILTER:       .where(Model.col == val)                         │
│  AND:          .where(col1 == x, col2 == y) or and_(...)        │
│  OR:           .where(or_(col1 == x, col2 == y))                │
│  IN:           .where(Model.col.in_([1, 2, 3]))                 │
│  LIKE:         .where(Model.col.contains("text"))               │
│  NULL:         .where(Model.col.is_(None))                      │
│  ORDER:        .order_by(Model.col.desc())                      │
│  LIMIT:        .limit(10)                                       │
│  OFFSET:       .offset(20)                                      │
│  JOIN:         .join(Model.relationship)                        │
│                                                                 │
│  EAGER LOADING (prevent N+1):                                   │
│    .options(joinedload(Model.rel))     → 1 JOIN query           │
│    .options(selectinload(Model.rel))   → 2 queries (IN)         │
│                                                                 │
│  REMEMBER:                                                      │
│    joinedload → needs .scalars().unique().all()                 │
│    selectinload → needs .scalars().all()                        │
│                                                                 │
│  PAGINATION:                                                    │
│    Offset: .order_by(id).offset(skip).limit(size)               │
│    Keyset: .where(id > cursor).order_by(id).limit(size)         │
│                                                                 │
│  COMMON MISTAKES:                                               │
│    ❌ Accessing .posts in a loop without eager loading → N+1    │
│    ❌ Accessing relationship after session closes → Detached    │
│    ❌ Forgetting back_populates on BOTH sides                   │
│    ❌ Forgetting .unique() with joinedload                      │
│    ❌ Using OFFSET for deep pages on large tables               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Models

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  RELATIONSHIPS = NAVIGABLE CONNECTIONS                          │
│                                                                 │
│  ForeignKey is the wire in the database.                        │
│  relationship() is the Python handle on that wire.              │
│  back_populates makes the handle work in both directions.       │
│                                                                 │
│  ┌────────┐  .posts   ┌────────┐  .tags   ┌────────┐           │
│  │ Author │──────────▶│  Post  │◀────────▶│  Tag   │           │
│  │        │◀──────────│        │          │        │           │
│  └────────┘  .author  └────────┘          └────────┘           │
│                                                                 │
│                                                                 │
│  N+1 = DEATH BY A THOUSAND QUERIES                              │
│                                                                 │
│  Every time you access a lazy relationship in a loop,           │
│  you make another phone call to the database.                   │
│  100 authors = 101 queries. 1000 authors = 1001 queries.        │
│                                                                 │
│  The fix: tell SQLAlchemy to load eagerly with .options():      │
│  joinedload  = one big JOINed query (small related sets)        │
│  selectinload = two efficient queries (large related sets)      │
│                                                                 │
│                                                                 │
│  PAGINATION = HOW YOU SERVE DATA IN CHUNKS                      │
│                                                                 │
│  Offset: simple, slow at depth.                                 │
│  Keyset: fast at any depth, but no random page access.          │
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
│  WEEK 6, LECTURE 3 (Alembic):                                   │
│  └─ When you add relationship() to a model, you may need       │
│     new columns (ForeignKey) or tables (association).           │
│     Alembic detects these changes and generates migrations.     │
│                                                                 │
│  WEEK 6, LECTURE 4 (Async SQLAlchemy):                          │
│  └─ Everything you learned today works with AsyncSession.       │
│     select(), .options(), joinedload — same API, async engine.  │
│     The N+1 problem is WORSE in async servers because           │
│     lazy queries block the event loop (Week 1, Lecture 3).      │
│                                                                 │
│  WEEK 6 PROJECT (Task Manager Refactor):                        │
│  └─ You'll implement these patterns: Tasks belong to Users,     │
│     Tasks have Tags (many-to-many), paginated endpoints.        │
│     This lecture is your toolkit.                               │
│                                                                 │
│  WEEK 7 (Query Optimization):                                   │
│  └─ EXPLAIN ANALYZE on your queries. Add indexes.               │
│     Measure joinedload vs selectinload performance.             │
│     Composite indexes for keyset pagination.                    │
│                                                                 │
│  WEEK 10 (Redis Caching):                                       │
│  └─ Cache expensive queries instead of hitting the database     │
│     every time. Pagination + caching = fast APIs.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```