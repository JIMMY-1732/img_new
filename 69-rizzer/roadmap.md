```
### PHASE 0: PYTHON FOUNDATIONS (Week 1-2)

├─ Week 1: Python for Backend Development
|    ├─ LECTURE 1: Type Hints for Backend
|    │   ├─ Type hint fundamentals (built-in types, Optional, Union)
|    │   ├─ Collection types (list, dict, set with type params)
|    │   ├─ Type aliases (clean complex signatures)
|    │   ├─ Introduction to generics (TypeVar basics — simple cases only)
|    │   └─ Running mypy (catching bugs before runtime)
|    │
|    ├─ LECTURE 2: Python Patterns for Backend
|    │   ├─ Dataclasses (foundation for Pydantic understanding)
|    │   ├─ Context managers (with statement, resource cleanup)
|    │   ├─ Decorators demystified (understand @syntax before FastAPI)
|    │   ├─ Custom exceptions and exception hierarchies
|    │   └─ Error handling patterns (try/except/finally, re-raising)
|    │
|    ├─ LECTURE 3: Async Fundamentals
|    │   ├─ Blocking vs non-blocking (the problem async solves)
|    │   ├─ Event loop mental model (single-threaded concurrency)
|    │   ├─ async/await syntax (coroutines, awaitables)
|    │   ├─ asyncio.gather and asyncio.create_task (concurrent execution)
|    │   ├─ When to use async (I/O-bound vs CPU-bound decision)
|    │   └─ Common pitfalls (blocking calls in async code)
|    │
|    ├─LECTURE 4: Developer Workflow
│    ├─ Git fundamentals (commits, branches, merge vs rebase)
│    ├─ Pull request workflow (code review culture)
│    ├─ Conventional commits (semantic versioning prep)
│    ├─ uv for project management (uv init, uv add, uv run, uv lock)
│    │   ├─ Python version management with uv (uv python install, uv python pin)
│    │   ├─ pyproject.toml as single source of truth (dependencies, metadata, config)
│    │   └─ uv.lock for reproducible environments (cross-platform lockfile)
│    └─ Legacy context: venv, pip, requirements.txt (what you'll encounter in existing codebases)


Week 2: Debugging, Testing Foundations & First Build

├─ LECTURE 1: Debugging Methodology
│   ├─ Reading stack traces (anatomy of a Python traceback)
│   ├─ IDE debugger as primary tool (VS Code / PyCharm)
│   │   ├─ Visual breakpoints, conditional breakpoints
│   │   ├─ Variable inspection pane and watch expressions
│   │   ├─ Step into, step over, step out (visual controls)
│   │   └─ Debug configurations for FastAPI / pytest
│   ├─ pdb and breakpoint() as universal fallback
│   │   ├─ When GUI isn't available (remote servers, SSH, Docker containers)
│   │   └─ Core commands: step, next, continue, inspect, where
│   ├─ Binary search debugging (isolating the failure)
│   ├─ Rubber duck debugging and hypothesis-driven debugging
│   ├─ Reading other people's code (strategies, not just skimming)
│   └─ When to ask for help (what information to include)
│
├─ LECTURE 2: Testing Fundamentals
│   ├─ Why test? (confidence, refactoring safety, documentation)
│   ├─ pytest fundamentals (test functions, assertions, parametrize)
│   ├─ Fixtures for test setup (@pytest.fixture, scope, conftest.py)
│   ├─ Unit vs integration vs end-to-end (testing pyramid)
│   ├─ Test doubles: mocks, stubs, fakes (unittest.mock basics)
│   └─ Test-driven development intro (red-green-refactor cycle)
│
└─ MINI-PROJECT: Async CLI Weather/News Aggregator
    Requirements: Fetch data from 3 public APIs concurrently
    (e.g., weather, news, exchange rates), proper error handling,
    type hints throughout, pytest suite with mocked API calls.

    WHY: Same async+httpx exercise as original, but domain-neutral.
    Tests are now required FROM THE FIRST PROJECT.


---

### PHASE 1: API FUNDAMENTALS (Week 3-4)


**Week 3: FastAPI Core**


├─ LECTURE 1: HTTP & REST Foundations
│   ├─ HTTP protocol basics (request/response cycle, headers, body)
│   ├─ HTTP methods semantics (GET, POST, PUT, PATCH, DELETE)
│   ├─ Status codes and their meanings (2xx, 4xx, 5xx families)
│   ├─ REST principles (resources, statelessness, uniform interface)
│   ├─ REST vs GraphQL vs gRPC (when each is appropriate) 
│   └─ JSON format (serialization basics)
│
├─ LECTURE 2: FastAPI Fundamentals
│   ├─ App initialization and ASGI (uvicorn)
│   ├─ Path operations (@app.get, @app.post, etc.)
│   ├─ Path parameters and type coercion
│   ├─ Query parameters (required, optional, defaults)
│   ├─ Request body basics (JSON input)
│   └─ Automatic OpenAPI documentation (Swagger UI, ReDoc)
│
├─ LECTURE 3: Pydantic Deep Dive
│   ├─ BaseModel fundamentals (schema as class)
│   ├─ Field constraints (Field, gt, lt, min_length, regex)
│   ├─ Nested models (composition)
│   ├─ Custom validators (@field_validator, @model_validator)
│   ├─ Model configuration (model_config, from_attributes)
│   ├─ Serialization control (exclude, include, by_alias)
│   └─ Request vs Response models (separation of concerns)
│
├─ LECTURE 4: Error Handling & Dependencies
│   ├─ HTTPException (status codes, detail messages)
│   ├─ Custom exception handlers (@app.exception_handler)
│   ├─ Dependency injection with Depends()
│   ├─ Yield dependencies (setup/teardown pattern)
│   ├─ Dependency scope and caching
│   └─ Basic logging setup (structlog introduction)
│
└─ PROJECT: In-Memory Task Manager API (Week 3-4)
    Endpoints: CRUD for tasks, categories, tags
    Requirements: Pydantic models, proper status codes,
    error handling, 80%+ test coverage, mocked unit tests


**Week 4: Testing APIs & API Design Patterns**


├─ LECTURE 1: Testing FastAPI Applications
│   ├─ TestClient for API testing (sync wrapper)
│   ├─ Testing different status codes and validation errors
│   ├─ Mocking dependencies in tests (dependency_overrides)
│   ├─ Testing async endpoints (pytest-asyncio, httpx.AsyncClient)
│   ├─ Integration tests vs unit tests for APIs
│   └─ Test organization (conftest.py, factories, fixtures)
│
├─ LECTURE 2: API Design Principles 
│   ├─ API versioning strategies (URL, header, query param)
│   ├─ Pagination design (offset vs cursor — concept first)
│   ├─ Filtering and sorting conventions
│   ├─ Idempotency in API design (why PUT and DELETE are idempotent)
│   ├─ HATEOAS awareness (links in responses)
│   └─ API documentation as contract (OpenAPI spec)
│
└─ Complete and deliver Task Manager API project
    Added requirement: API versioning, cursor pagination,
    comprehensive test suite (unit + integration)


---

### PHASE 2: DATABASES & PERSISTENCE (Week 5-7)


**Week 5: SQL & PostgreSQL Fundamentals**


├─ LECTURE 1: Relational Database Concepts
│   ├─ Why relational databases? (ACID, data integrity, relationships)
│   ├─ Tables, rows, columns, primary keys, foreign keys
│   ├─ Data types in PostgreSQL (TEXT, INTEGER, BOOLEAN, TIMESTAMP, UUID, JSONB)
│   ├─ Normalization basics (1NF, 2NF, 3NF — when and why)
│   ├─ Denormalization tradeoffs (when to break the rules)
│   └─ Entity-Relationship diagrams (designing before coding)
│
├─ LECTURE 2: SQL Fundamentals
│   ├─ Running PostgreSQL with Docker [Docker introduced early as a tool]
│   ├─ SELECT, INSERT, UPDATE, DELETE (CRUD in SQL)
│   ├─ WHERE, ORDER BY, LIMIT, OFFSET
│   ├─ JOINs (INNER, LEFT, RIGHT, FULL — with visual mental models)
│   ├─ Aggregate functions (COUNT, SUM, AVG, GROUP BY, HAVING)
│   └─ Subqueries and CTEs (Common Table Expressions)
│
├─ LECTURE 3: PostgreSQL-Specific Features
│   ├─ JSONB columns (semi-structured data in a relational DB)
│   ├─ Array types
│   ├─ Full-text search basics (tsvector, tsquery)
│   ├─ EXPLAIN basics (how to read a query plan — introduction)
│   ├─ Indexes introduction (B-tree default, when to add them)
│   └─ Transactions (BEGIN, COMMIT, ROLLBACK, isolation levels intro)
│
├─ LECTURE 4: Database Design Workshop
│   ├─ Designing schemas from requirements (guided exercise)
│   ├─ One-to-many, many-to-many relationships
│   ├─ Junction tables for many-to-many
│   ├─ Choosing between UUID and serial PKs
│   ├─ Soft deletes vs hard deletes
│   └─ Timestamps (created_at, updated_at patterns)
│
└─ MINI-PROJECT: Design and populate a database
    Requirements: Design schema for a multi-entity system
    (e.g., users, organizations, projects, tasks), write raw SQL
    queries, document schema with ER diagram


**Week 6: SQLAlchemy & Alembic**


├─ LECTURE 1: SQLAlchemy ORM Fundamentals
│   ├─ ORM concept (mapping objects to tables — the why)
│   ├─ SQLAlchemy 2.0 style (Mapped, mapped_column)
│   ├─ Engine and Session (connection lifecycle)
│   ├─ Defining models (DeclarativeBase, table metadata)
│   ├─ Column types and constraints
│   └─ Creating tables (metadata.create_all — development only)
│
├─ LECTURE 2: Relationships & Querying
│   ├─ Relationship definitions (one-to-many, many-to-many)
│   ├─ Back-populates and foreign keys
│   ├─ Basic queries (select, where, order_by, limit)
│   ├─ Filtering patterns (and_, or_, in_)
│   ├─ Eager vs lazy loading (the N+1 problem, joinedload, selectinload)
│   └─ Pagination with SQLAlchemy (offset and keyset-based)
│
├─ LECTURE 3: Alembic Migrations
│   ├─ Why migrations? (schema evolution without data loss)
│   ├─ Alembic setup and configuration
│   ├─ Autogenerate migrations (alembic revision --autogenerate)
│   ├─ Manual migrations (data migrations, complex changes)
│   ├─ Migration best practices (review before applying, CI checks)
│   └─ Rollback strategies (downgrade, testing migrations)
│
├─ LECTURE 4: Async SQLAlchemy & FastAPI Integration
│   ├─ AsyncSession and async engine (asyncpg driver)
│   ├─ SQLAlchemy as FastAPI dependency (session per request)
│   ├─ Repository pattern (separating DB logic from routes)
│   ├─ Pydantic + SQLAlchemy (from_attributes, response models)
│   ├─ Transaction management in request lifecycle
│   └─ Testing with database (test database, fixtures, rollback)
│
└─ PROJECT: Refactor Task Manager to use PostgreSQL
    Requirements: Full database-backed CRUD, Alembic migrations,
    async SQLAlchemy, repository pattern, test suite with
    test database (not mocks for integration tests)


**Week 7: Advanced Database Patterns**


├─ LECTURE 1: Query Optimization
│   ├─ EXPLAIN and EXPLAIN ANALYZE (reading query plans in depth)
│   ├─ Index types (B-tree, Hash, GIN, GiST — when to use each)
│   ├─ Composite indexes (column order matters)
│   ├─ Partial indexes and expression indexes
│   ├─ Index-only scans
│   └─ Query profiling with SQLAlchemy (echo, logging)
│
├─ LECTURE 2: Advanced Patterns
│   ├─ Connection pooling (pool_size, max_overflow, pool_recycle)
│   ├─ Optimistic vs pessimistic locking
│   ├─ Bulk operations (SQLAlchemy 2.0 style)
│   │   ├─ session.execute(insert(Model), [...]) for bulk INSERT
│   │   ├─ session.execute(update(Model), [...]) for bulk UPDATE by primary key
│   │   ├─ session.add_all() for ORM-managed bulk inserts
│   │   └─ Legacy awareness: bulk_save_objects/insert_mappings (deprecated, encounter in old code)
│   ├─ Raw SQL when ORM isn't enough (text(), hybrid properties)
│   ├─ Multi-tenancy patterns (schema-based, row-based)
│   └─ Database seeding and fixtures for development
│
├─ LECTURE 3: NoSQL Awareness
│   ├─ When relational isn't the answer (document stores, key-value)
│   ├─ MongoDB overview (documents, collections, when to use)
│   ├─ Redis as more than cache (data structures, pub/sub — preview)
│   ├─ Choosing the right database for the job (decision framework)
│   └─ Polyglot persistence (using multiple databases together)
│
└─ Complete database-backed Task Manager with optimized queries
    Added requirement: at least 2 queries with EXPLAIN showing
    index usage, documented before/after performance numbers



---

### PHASE 3: EXTERNAL INTEGRATION & AUTHENTICATION (Week 8-9)


**Week 8: External APIs & HTTP Clients**


├─ LECTURE 1: HTTP Client Fundamentals
│   ├─ httpx overview (sync and async in one library)
│   ├─ Making GET and POST requests
│   ├─ Handling response data (JSON parsing, status checking)
│   ├─ Timeouts configuration (connect, read, write, pool)
│   ├─ Retry strategies (tenacity library, exponential backoff)
│   └─ Connection pooling with AsyncClient (reuse connections)
│
├─ LECTURE 2: External API Patterns
│   ├─ API authentication methods (API keys, OAuth, headers)
│   ├─ Rate limiting awareness (respecting X-RateLimit headers)
│   ├─ Client-side rate limiting (token bucket algorithm)
│   ├─ Pagination handling (cursor-based, offset-based)
│   ├─ Webhook basics (receiving external events)
│   └─ Circuit breaker pattern (fail fast, recover gracefully)
│
├─ LECTURE 3: Data Transformation & Integration
│   ├─ External vs internal models (never trust external data)
│   ├─ Pydantic for API response validation
│   ├─ Handling missing/extra fields gracefully
│   ├─ Data normalization (different sources, same internal format)
│   ├─ Storing external data in your database (sync strategies)
│   └─ Background data refresh (FastAPI BackgroundTasks intro)
│
└─ PROJECT: Third-Party Integration Service
    Features: Integrate with 2-3 real public APIs (e.g., GitHub API,
    OpenWeatherMap, a payment provider sandbox), normalize data,
    store in PostgreSQL, expose unified API.
    Requirements: Retry logic, circuit breaker, rate limit compliance,
    webhook endpoint for at least one service.


**Week 9: Authentication & Security**


├─ LECTURE 1: Authentication Fundamentals
│   ├─ Authentication vs Authorization (definitions)
│   ├─ Password hashing (why not MD5/SHA — preimage and rainbow table attacks)
│   ├─ Modern hashing with pwdlib
│   │   ├─ PasswordHash.recommended() (defaults to Argon2id)
│   │   ├─ Why Argon2id (memory-hard, winner of Password Hashing Competition)
│   │   ├─ Bcrypt as still-secure alternative (when you'll encounter it)
│   │   └─ Legacy awareness: passlib (unmaintained, broken on Python 3.13+/bcrypt 5.x)
│   ├─ User registration flow (validation, duplicate check, DB storage)
│   ├─ Login flow (credential verification)
│   └─ Secure password requirements
│
├─ LECTURE 2: JWT Authentication
│   ├─ Token-based auth vs session-based (tradeoffs)
│   ├─ JWT structure (header, payload, signature)
│   ├─ Access tokens (short-lived) and refresh tokens (rotation)
│   ├─ PyJWT for JWT operations
│   │   ├─ jwt.encode() and jwt.decode() (simple, focused API)
│   │   ├─ Algorithm selection (HS256 symmetric, RS256 asymmetric)
│   │   └─ Legacy awareness: python-jose (historically used, FastAPI docs moved to PyJWT)
│   ├─ Token expiration and validation
│   └─ Storing tokens client-side (security considerations)
│
├─ LECTURE 3: Protected Endpoints & RBAC
│   ├─ OAuth2PasswordBearer (FastAPI's auth scheme)
│   ├─ get_current_user dependency (token → user from DB)
│   ├─ Role-based access control (user roles, permissions)
│   ├─ OAuth2 scopes in FastAPI (fine-grained permissions)
│   ├─ Admin-only endpoints (role checking dependency)
│   └─ Optional authentication (public + auth endpoints)
│
├─ LECTURE 4: API Security
│   ├─ CORS configuration (origins, methods, headers, middleware)
│   ├─ SQL injection prevention (parameterized queries — with ORM context)
│   ├─ Input validation as security (Pydantic's role)
│   ├─ Rate limiting for auth endpoints (brute force prevention)
│   ├─ Security headers (X-Content-Type-Options, etc.)
│   └─ OWASP Top 10 awareness (what to watch for)
│
└─ PROJECT: Add Auth to Task Manager / Integration Service
    Features: User registration, login, protected resources,
    user-specific data isolation (DB-backed), admin management.
    Requirements: JWT with refresh, RBAC, CORS configured,
    rate limiting on login, all stored in PostgreSQL.


---

### PHASE 4: CACHING, PERFORMANCE & BACKGROUND JOBS (Week 10-12)


**Week 10: Redis & Caching**


├─ LECTURE 1: Redis Fundamentals
│   ├─ Redis overview (in-memory data store, use cases)
│   ├─ Redis CLI basics (SET, GET, DEL, KEYS, TTL)
│   ├─ Data types: Strings (values, counters with INCR)
│   ├─ Data types: Hashes (object storage)
│   ├─ Data types: Lists (queues, recent items)
│   ├─ Data types: Sets and Sorted Sets (rankings, unique items)
│   └─ TTL (Time-To-Live) for automatic expiration
│
├─ LECTURE 2: Redis in Python & Caching Patterns
│   ├─ redis.asyncio client (async-first)
│   ├─ Connection pooling (async pools)
│   ├─ Redis as FastAPI dependency
│   ├─ Cache-aside pattern (check cache, fallback to DB)
│   ├─ Cache invalidation strategies (time-based, event-based)
│   ├─ Cache key design (namespacing, versioning)
│   └─ Cache stampede prevention (locking, probabilistic expiry)
│
├─ LECTURE 3: Session & Token Storage
│   ├─ Storing refresh tokens in Redis (fast lookup + TTL)
│   ├─ Token revocation (logout, security events)
│   ├─ Distributed rate limiting with Redis (INCR, EXPIRE)
│   ├─ Session data storage patterns
│   └─ Graceful degradation (what happens when Redis is down)
│
└─ PROJECT: Add Caching Layer
    Features: Cache expensive DB queries, cache external API responses,
    implement TTL, cache invalidation on write, hit rate monitoring.
    Requirements: Async Redis, proper key design,
    graceful degradation if Redis unavailable.


**Week 11: Background Jobs & Event-Driven Patterns**


├─ LECTURE 1: Background Processing Concepts
│   ├─ Why background jobs? (long-running, scheduled, decoupled)
│   ├─ FastAPI BackgroundTasks (fire-and-forget, limitations)
│   ├─ Message broker pattern (producer, broker, consumer)
│   ├─ Job queue vs pub/sub (different use cases) [CONCEPT FIRST]
│   └─ Event-driven architecture overview [NEW — concept, not tool]
│
├─ LECTURE 2: Celery Fundamentals
│   ├─ Celery + Redis setup (broker and result backend)
│   ├─ Defining tasks (@celery.task decorator)
│   ├─ Calling tasks (delay, apply_async)
│   ├─ Task results (AsyncResult, task states)
│   ├─ Error handling, retries (autoretry_for, retry_backoff)
│   └─ Idempotency (why and how to make tasks idempotent)
│
├─ LECTURE 3: Scheduling & Monitoring
│   ├─ Celery Beat for periodic tasks (crontab schedules)
│   ├─ Task timeouts (soft_time_limit, time_limit)
│   ├─ Dead letter handling (poison messages)
│   ├─ Flower for monitoring (web UI, task history)
│   └─ Alerting on task failures (logging, health checks)
│
├─ LECTURE 4: Event-Driven Architecture & Task Queue Selection
│   ├─ Pub/Sub pattern (decouple producers from consumers)
│   ├─ Redis Pub/Sub (simple event broadcasting)
│   ├─ When you'd need Kafka/RabbitMQ (and when you don't)
│   ├─ Event sourcing awareness (concept, not implementation)
│   ├─ CQRS awareness (read/write separation concept)
│   └─ Choosing the right tool for the job
│       ├─ FastAPI BackgroundTasks (simple fire-and-forget, no persistence)
│       ├─ Celery (industry standard, battle-tested, sync-first architecture)
│       ├─ Taskiq (async-native Celery alternative, FastAPI integration, type-hinted)
│       ├─ ARQ (async + Redis only, pessimistic execution, at-least-once delivery)
│       └─ Decision framework: complexity vs async-nativeness vs ecosystem maturity
│
└─ PROJECT: Background Processing Pipeline
    Features: Scheduled data processing jobs, event-triggered
    notifications (e.g., email on user action), retry logic.
    Requirements: Idempotent tasks, monitoring, graceful shutdown.


**Week 12: WebSockets, Real-Time & Performance**


├─ LECTURE 1: WebSocket Fundamentals
│   ├─ HTTP vs WebSocket (request-response vs bidirectional)
│   ├─ WebSocket lifecycle (open, message, close, error)
│   ├─ FastAPI WebSocket endpoints (@app.websocket)
│   ├─ Connection manager pattern (tracking active connections)
│   ├─ Broadcasting and room/channel patterns
│   └─ WebSocket authentication (token in query param or first message)
│
├─ LECTURE 2: Real-Time Patterns
│   ├─ Heartbeat/ping-pong (dead connection detection)
│   ├─ Reconnection strategies (client-side awareness)
│   ├─ Scaling WebSockets (the problem, Redis pub/sub solution)
│   ├─ Server-Sent Events as alternative (SSE, simpler for push-only)
│   └─ Testing WebSockets (pytest patterns)
│
├─ LECTURE 3: Performance Measurement & Optimization
│   ├─ Profiling mindset (measure BEFORE optimizing)
│   ├─ Request timing middleware (log slow requests)
│   ├─ Database query profiling (N+1 detection in production)
│   ├─ Response compression (GzipMiddleware)
│   ├─ Async endpoint optimization (gather for parallel operations)
│   └─ Load testing with locust (write scenarios, interpret results)
│
├─ LECTURE 4: Rate Limiting Your API
│   ├─ Why rate limit (fairness, protection, cost control)
│   ├─ slowapi implementation (limits, key functions)
│   ├─ Rate limit headers (X-RateLimit-*)
│   ├─ Performance regression testing in CI
│   └─ Interpreting load test results (p50, p95, p99 latency)
│
└─ PROJECT: Add Real-Time & Optimize
    Target: Handle 500 req/min with <200ms p95 latency.
    Features: WebSocket notifications for real-time updates,
    background job triggers, cached responses.
    Requirements: Load test report with before/after metrics,
    at least 3 documented optimizations.



---

### PHASE 5: CAPSTONE PROJECT (Week 13-14) 


**Week 13-14: Multi-Tenant SaaS Platform Backend**


├─ LECTURE 1: Project Architecture & Domain Modeling
│   ├─ Requirements analysis (translating business needs to technical spec)
│   ├─ Domain modeling exercise (entities, relationships, boundaries)
│   ├─ Architecture Decision Records (ADRs — document your choices) [NEW]
│   ├─ Project structure for larger FastAPI applications
│   │   (routers, services, repositories, schemas, models)
│   └─ API contract design (OpenAPI spec before implementation)
│
├─ LECTURE 2: Multi-Tenancy & Advanced Patterns
│   ├─ Multi-tenancy strategies (row-level isolation, schema-per-tenant)
│   ├─ Data isolation enforcement (middleware, dependencies)
│   ├─ Shared resources vs tenant-specific resources
│   ├─ Audit logging (who did what, when)
│   └─ Soft deletes and data retention policies
│
├─ LECTURE 3: Integration Patterns
│   ├─ Email/notification service integration (SendGrid/webhook pattern)
│   ├─ File upload handling (S3-compatible storage, presigned URLs)
│   ├─ Search implementation (PostgreSQL full-text or simple LIKE)
│   ├─ Export functionality (CSV/JSON generation as background task)
│   └─ Admin dashboard API (aggregation queries, statistics)
│
├─ LECTURE 4: Code Review & Technical Communication
│   ├─ Code review best practices (what to look for, how to give feedback)
│   ├─ Writing meaningful PR descriptions
│   ├─ Technical documentation (README, API docs, runbooks)
│   ├─ Estimation basics (breaking down tasks, time vs complexity)
│   └─ Working with existing codebases (reading before writing)
│
└─ CAPSTONE PROJECT: Project Management SaaS Backend
    (or choose: invoice management, team collaboration tool,
    content management platform — any multi-entity SaaS domain)

    Core features:
    ├─ User authentication with JWT (registration, login, refresh)
    ├─ Multi-tenant organization support (users belong to orgs)
    ├─ CRUD for domain entities (projects, tasks, members, etc.)
    ├─ Role-based access control (admin, member, viewer per org)
    ├─ Real-time notifications via WebSocket (task updates, mentions)
    ├─ Background jobs (email notifications, report generation)
    ├─ Redis caching for frequently accessed data
    ├─ Search and filtering with pagination
    ├─ File attachment support
    └─ Audit log for all mutations

    Requirements:
    ├─ PostgreSQL with Alembic migrations
    ├─ Async SQLAlchemy throughout
    ├─ Comprehensive test suite (unit + integration, 80%+ coverage)
    ├─ API documentation via OpenAPI
    ├─ Architecture Decision Records
    ├─ Load tested with documented results
    └─ Clean git history with conventional commits


---

### PHASE 6: PRODUCTION & DEPLOYMENT (Week 15-16)


**Week 15: Containerization, CI/CD & Cloud Basics**


├─ LECTURE 1: Containerization Concepts & Docker
│   ├─ What is a container? (process isolation, namespaces, cgroups — conceptual)
│   ├─ Containers vs VMs (tradeoffs, when each is appropriate)
│   ├─ Dockerfile best practices (multi-stage builds, layer caching)
│   ├─ Docker Compose for multi-service (API, worker, Redis, Postgres)
│   ├─ Health checks and service dependencies
│   └─ Volume management and networking basics
│
├─ LECTURE 2: Configuration, Secrets & Observability
│   ├─ 12-factor app principles (config in environment)
│   ├─ pydantic-settings for configuration validation
│   ├─ Secrets management (never commit secrets, env vars, .env files)
│   ├─ Structured logging (JSON logs, correlation IDs)
│   ├─ Health check endpoints (/health, /ready)
│   └─ Error tracking (Sentry integration)
│
├─ LECTURE 3: CI/CD Pipelines
│   ├─ GitHub Actions fundamentals (workflows, jobs, steps)
│   ├─ CI pipeline
│   │   ├─ Lint with ruff check (replaces Flake8, isort, pyupgrade, and dozens of plugins)
│   │   ├─ Format check with ruff format --check (replaces Black — single tool for both)
│   │   ├─ Type check with mypy --strict
│   │   ├─ Test with pytest and coverage reporting
│   │   └─ Why one tool: ruff handles linting + formatting + import sorting in a single pass
│   ├─ CD pipeline (build Docker image, push to registry, deploy)
│   ├─ Database migrations in CI/CD (Alembic in deployment pipeline)
│   ├─ Environment-specific configuration (dev, staging, prod)
│   └─ Rolling deployments and rollback strategies
│
├─ LECTURE 4: Cloud Fundamentals [NEW]
│   ├─ Cloud provider overview (AWS/GCP/Azure — pick one to learn)
│   ├─ Core services map: compute (EC2/Cloud Run), database (RDS),
│   │   storage (S3), networking (VPC basics), IAM (permissions)
│   ├─ Managed vs self-hosted (when to use RDS vs Docker Postgres)
│   ├─ Infrastructure as Code awareness (Terraform concepts)
│   ├─ Cost awareness (how cloud billing works, avoiding surprises)
│   └─ Serverless awareness (Lambda/Cloud Functions — when appropriate)
│
└─ DELIVERABLE: Dockerized application with CI/CD pipeline
    Requirements: Docker Compose for local dev,
    GitHub Actions CI (lint + type check + test + coverage),
    deployed to a platform (student choice: Railway, Fly.io, or AWS)


**Week 16: System Design, Career Prep & Final Delivery**


├─ LECTURE 1: System Design Fundamentals
│   ├─ System design interview format (requirements → design → tradeoffs)
│   ├─ Load balancers (distributing traffic)
│   ├─ Horizontal vs vertical scaling
│   ├─ Database scaling (read replicas, sharding concepts)
│   ├─ Caching layers (CDN, application cache, database cache)
│   └─ CAP theorem and real-world implications
│
├─ LECTURE 2: Architecture Patterns
│   ├─ Monolith vs microservices (tradeoffs, when to split)
│   ├─ Message queues in architecture (decoupling services)
│   ├─ Event-driven architecture at scale (Kafka/RabbitMQ in system design)
│   ├─ API gateway pattern
│   ├─ Service mesh awareness
│   └─ Designing for failure (redundancy, fallbacks, graceful degradation)
│
├─ LECTURE 3: DSA & Interview Prep Strategy [NEW]
│   ├─ DSA practice strategy (what to focus on for backend roles)
│   ├─ Common backend interview patterns (hash maps, queues, trees, graphs)
│   ├─ Time/space complexity analysis applied to your own code
│   ├─ SQL interview questions (common patterns, window functions)
│   ├─ Behavioral interview preparation (STAR method)
│   └─ Portfolio presentation (how to talk about your projects)
│
└─ FINAL DELIVERABLE: Production-Ready SaaS Backend
    Requirements:
    ├─ Docker Compose for full local environment
    ├─ CI/CD pipeline with GitHub Actions
    ├─ Deployed to cloud platform
    ├─ Structured logging, health checks, error tracking
    ├─ README with architecture diagram
    ├─ Architecture Decision Records
    ├─ Demo video / live presentation
    └─ Post-mortem document: what you'd do differently


---

### ONGOING (Parallel Track, Weeks 5-16)


DSA PRACTICE (not a phase — a daily habit)
├─ Start Week 5 (once API fundamentals are solid)
├─ 30 minutes/day, 3-5 problems/week
├─ Focus areas for backend roles:
│   ├─ Hash maps and sets (most common in backend interviews)
│   ├─ Queues and stacks (BFS, task scheduling)
│   ├─ Trees (hierarchical data, file systems)
│   ├─ Graphs (dependency resolution, network topology)
│   ├─ SQL problems (LeetCode SQL track, HackerRank SQL)
│   └─ System design practice (start Week 12)
├─ Platform: LeetCode or NeetCode (curated lists, not random)
└─ Track: solve at minimum 75 problems by Week 16
```