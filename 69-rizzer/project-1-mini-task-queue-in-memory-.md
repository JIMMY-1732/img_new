# Week 1 Project: Async Service Health Checker

## The Scenario

You've joined the platform reliability team. The team maintains a tool that monitors whether internal services (`user-service`, `payment-service`, etc.) are alive and responding.

The current script checks each service one at a time. With 20+ services, a full health check cycle takes over a minute. This is too slow for real-time alerting.

**Your Mission:** Your tech lead wants a prototype that checks all services **concurrently**, handles failures gracefully, retries flaky services, and reports structured results — all with strict type safety.

---

## Getting Started

### Prerequisites

- **Python 3.11 or newer.** Verify with `python --version`.
- **Git** installed and configured with your name/email.

### Setup Steps

Every command below is run from your terminal. If anything fails, **read the error message** before asking for help — the message usually tells you what's wrong.

```bash
# 1. Clone the starter repository
git clone <REPO_URL> week1-health-checker
cd week1-health-checker

# 2. Create a feature branch (do NOT work on main)
git checkout -b feat/health-checker

# 3. Create and activate a virtual environment
python -m venv .venv

# On macOS / Linux:
source .venv/bin/activate

# On Windows (PowerShell):
.venv\Scripts\Activate.ps1

# 4. Install the project in development mode
pip install -e ".[dev]"

# 5. Verify: run the tests (they should ALL FAIL — that's expected)
pytest tests/ -v

# 6. Verify: run the type checker (it should report errors — that's expected)
mypy src/
```

After step 5, you should see red. Every test failing means the harness works and your job is to turn them green, one file at a time.

---

## Repository Structure

Files marked ✅ are **pre-written — do not modify them.**
Files marked ✏️ are **yours to implement.**

```
week1-health-checker/
│
├── pyproject.toml                ✅  Project config, dependencies, tool settings
├── .gitignore                    ✅  Keeps junk out of your repo
├── README.md                     ✅  This document
├── services.json                 ✅  Sample service configuration for main.py
│
├── src/
│   ├── __init__.py               ✅  Package marker
│   ├── _simulator.py             ✅  Simulates HTTP responses (DO NOT MODIFY)
│   ├── models.py                 ✏️  Your data models (ServiceConfig, HealthResult)
│   ├── exceptions.py             ✏️  Your exception hierarchy
│   ├── retry.py                  ✏️  Your retry decorator
│   ├── checker.py                ✏️  Your async health-checking logic
│   └── main.py                   ✏️  Entry point — run a full health check
│
└── tests/
    ├── conftest.py               ✅  Shared test configuration
    ├── test_models.py            ✅  Tests for your data models
    ├── test_exceptions.py        ✅  Tests for your exception hierarchy
    ├── test_retry.py             ✅  Tests for your retry decorator
    └── test_checker.py           ✅  Tests for your async checker
```

---

## About the Simulator (`_simulator.py`)

This module simulates HTTP service responses using `asyncio.sleep` for realistic delays. **You do not need any HTTP library** (`httpx`, `requests`, `aiohttp`) for this project.

The simulator provides two things you will use in `checker.py`:

**`SimulatedResponse`** — A frozen dataclass with two fields: `status_code: int` and `body: str`. This represents what a service sends back.

**`SimulatedSession`** — Simulates an HTTP client session. It **must** be used as an async context manager (`async with`). It has one method: `request(url: str) -> SimulatedResponse`.

How the simulator decides what to return is based on keywords in the URL:

| URL contains | Simulator behavior |
| :--- | :--- |
| `"healthy"` | Returns `200 OK` after 0.3–0.7s (random delay) |
| `"unhealthy"` | Returns `500 Internal Server Error` after \~0.1s |
| `"unreachable"` | Raises `ConnectionError` immediately |
| `"slow"` | Sleeps for 15 seconds (will exceed your timeout) |

The simulator is deterministic enough for tests, random enough to feel realistic. Read its source code — it's short, well-commented, and shows patterns you'll use yourself.

---

## How to Read Tests as Specifications

This is a skill you will use throughout your career. The pre-written tests are not just grading tools — they are your **executable specification**. Before writing any code, open each test file and extract three things:

**1. What names must exist?** Look at the `import` lines at the top.
```python
from src.models import ServiceConfig, HealthResult
```
This tells you: `models.py` must export classes named `ServiceConfig` and `HealthResult`.

**2. What signatures must functions have?** Look at how tests call your code.
```python
config = ServiceConfig(name="x", url="http://...", expected_status=200, timeout=5.0)
```
This tells you: `ServiceConfig.__init__` must accept these four keyword arguments.

**3. What behavior is expected?** Look at the `assert` lines.
```python
assert result.is_healthy is True
assert result.response_time_ms > 0
```
This tells you: `HealthResult` must have fields `is_healthy` (bool) and `response_time_ms` (a positive number when the service responded).

**Read every test file before writing a single line of implementation code.** The tests contain all the answers about *what* to build. Your job is to figure out *how*.

---

## Pre-Written Files (Complete Source)

### `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=68.0"]
build-backend = "setuptools.backends._legacy:_Backend"

[project]
name = "week1-health-checker"
version = "0.1.0"
description = "Week 1 Project — Async Service Health Checker"
requires-python = ">=3.11"
dependencies = []

[project.optional-dependencies]
dev = [
    "pytest>=7.4",
    "pytest-asyncio>=0.23",
    "mypy>=1.8",
]

[tool.setuptools.packages.find]
where = ["."]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
pythonpath = ["."]

[tool.mypy]
strict = true
packages = ["src"]
```

### `.gitignore`

```gitignore
__pycache__/
*.py[cod]
*$py.class
*.egg-info/
dist/
build/
.eggs/
*.egg
.mypy_cache/
.pytest_cache/
.venv/
venv/
env/
.idea/
.vscode/
```

### `services.json`

```json
[
    {
        "name": "user-service",
        "url": "http://mock:8001/healthy",
        "expected_status": 200,
        "timeout": 5.0
    },
    {
        "name": "payment-service",
        "url": "http://mock:8002/healthy",
        "expected_status": 200,
        "timeout": 3.0
    },
    {
        "name": "notification-service",
        "url": "http://mock:8003/unhealthy",
        "expected_status": 200,
        "timeout": 5.0
    },
    {
        "name": "legacy-auth-service",
        "url": "http://mock:8004/slow",
        "expected_status": 200,
        "timeout": 2.0
    },
    {
        "name": "abandoned-service",
        "url": "http://mock:8005/unreachable",
        "expected_status": 200,
        "timeout": 5.0
    }
]
```

### `src/__init__.py`

```python
"""Week 1 Health Checker package."""
```

### `src/_simulator.py`

```python
"""
Service Response Simulator — DO NOT MODIFY THIS FILE
=====================================================
Simulates HTTP service responses so you can focus on async patterns,
error handling, and type safety without needing a real HTTP library.

Usage in your checker.py:

    from src._simulator import SimulatedSession

    async with SimulatedSession() as session:
        response = await session.request("http://mock:8001/healthy")
        print(response.status_code)  # 200
        print(response.body)         # "OK"

Behaviour is determined by keywords in the URL:
    "healthy"     → 200 OK after 0.3–0.7s random delay
    "unhealthy"   → 500 Internal Server Error after ~0.1s
    "unreachable" → raises ConnectionError immediately
    "slow"        → sleeps for 15s (will exceed most timeouts)
"""

from __future__ import annotations

import asyncio
import random
from dataclasses import dataclass
from types import TracebackType


@dataclass(frozen=True)
class SimulatedResponse:
    """Represents an HTTP response from a simulated service."""

    status_code: int
    body: str


class SimulatedSession:
    """
    Simulates an HTTP client session.

    MUST be used as an async context manager — calling .request()
    without opening the session first will raise RuntimeError.

    This mirrors real HTTP libraries: you open a session, make
    requests through it, then close it to release resources.
    """

    def __init__(self) -> None:
        self._opened: bool = False
        self._closed: bool = False

    async def __aenter__(self) -> SimulatedSession:
        self._opened = True
        return self

    async def __aexit__(
        self,
        exc_type: type[BaseException] | None,
        exc_val: BaseException | None,
        exc_tb: TracebackType | None,
    ) -> None:
        self._closed = True

    @property
    def is_closed(self) -> bool:
        """True if the session has been properly closed."""
        return self._closed

    async def request(self, url: str) -> SimulatedResponse:
        """
        Simulate an HTTP GET request to the given URL.

        Raises:
            RuntimeError: If the session was not opened with ``async with``.
            ConnectionError: If the URL contains ``"unreachable"``.

        Returns:
            SimulatedResponse with status_code and body.
        """
        if not self._opened:
            raise RuntimeError(
                "Session not opened. Use 'async with SimulatedSession() as session:'"
            )
        if self._closed:
            raise RuntimeError("Session is already closed.")

        # --- Unreachable services: immediate failure ---
        if "unreachable" in url:
            raise ConnectionError(f"Cannot connect to {url}")

        # --- Slow services: extreme latency ---
        if "slow" in url:
            await asyncio.sleep(15.0)
            return SimulatedResponse(status_code=200, body="OK (eventually)")

        # --- Unhealthy services: fast but bad status ---
        if "unhealthy" in url:
            await asyncio.sleep(0.1)
            return SimulatedResponse(status_code=500, body="Internal Server Error")

        # --- Default: healthy service with realistic delay ---
        delay = 0.3 + random.random() * 0.4  # 0.3 – 0.7 seconds
        await asyncio.sleep(delay)
        return SimulatedResponse(status_code=200, body="OK")
```

### `tests/conftest.py`

```python
"""Shared test configuration for Week 1 Health Checker."""
```

---

## Pre-Written Tests (Complete Source)

**These are the full test files in your repo.** Every test is shown. Run them with:

```bash
# Run all tests with verbose output
pytest tests/ -v

# Run a single test file
pytest tests/test_models.py -v

# Run a specific test by name
pytest tests/ -v -k "test_service_config_has_required_fields"
```

### `tests/test_models.py`

```python
"""
Tests for data models.
======================
Read every test here before opening models.py.
The constructor arguments and assert statements tell you
exactly what fields your dataclasses need and what types they hold.
"""

import dataclasses

from src.models import HealthResult, ServiceConfig


class TestServiceConfig:
    """ServiceConfig holds the information needed to check one service."""

    def test_has_required_fields(self) -> None:
        config = ServiceConfig(
            name="user-service",
            url="http://mock:8001/health",
            expected_status=200,
            timeout=5.0,
        )
        assert config.name == "user-service"
        assert config.url == "http://mock:8001/health"
        assert config.expected_status == 200
        assert config.timeout == 5.0

    def test_is_a_dataclass(self) -> None:
        assert dataclasses.is_dataclass(ServiceConfig)

    def test_is_not_a_plain_dict(self) -> None:
        """We want real dataclasses, not dicts pretending to be models."""
        config = ServiceConfig(
            name="user-service",
            url="http://mock:8001/health",
            expected_status=200,
            timeout=5.0,
        )
        assert not isinstance(config, dict)


class TestHealthResult:
    """HealthResult holds the outcome of checking one service."""

    def test_healthy_result(self) -> None:
        result = HealthResult(
            service_name="user-service",
            is_healthy=True,
            response_time_ms=142.7,
            status_code=200,
            error=None,
        )
        assert result.service_name == "user-service"
        assert result.is_healthy is True
        assert result.response_time_ms == 142.7
        assert result.status_code == 200
        assert result.error is None

    def test_unhealthy_result(self) -> None:
        result = HealthResult(
            service_name="broken-service",
            is_healthy=False,
            response_time_ms=None,
            status_code=None,
            error="Connection refused",
        )
        assert result.is_healthy is False
        assert result.error is not None
        assert result.response_time_ms is None
        assert result.status_code is None

    def test_is_a_dataclass(self) -> None:
        assert dataclasses.is_dataclass(HealthResult)

    def test_optional_fields_accept_none(self) -> None:
        """
        response_time_ms, status_code, and error must all accept None.
        Think about which type hint allows a value OR None.
        """
        result = HealthResult(
            service_name="some-service",
            is_healthy=False,
            response_time_ms=None,
            status_code=None,
            error=None,
        )
        assert result.response_time_ms is None
        assert result.status_code is None
        assert result.error is None

    def test_unhealthy_with_status_code(self) -> None:
        """A service can respond (status_code present) but still be unhealthy."""
        result = HealthResult(
            service_name="sick-service",
            is_healthy=False,
            response_time_ms=55.0,
            status_code=500,
            error="Expected 200, got 500",
        )
        assert result.is_healthy is False
        assert result.status_code == 500
        assert result.response_time_ms is not None
        assert result.error is not None
```

### `tests/test_exceptions.py`

```python
"""
Tests for exception hierarchy.
===============================
Your exceptions must form a tree:

    Exception  (built-in)
    └── HealthCheckError  (your base)
        ├── ServiceUnreachableError
        └── HealthCheckTimeoutError

Each exception stores context about WHICH service failed and WHY.
"""

import pytest

from src.exceptions import (
    HealthCheckError,
    HealthCheckTimeoutError,
    ServiceUnreachableError,
)


class TestHierarchy:
    """Verify the inheritance chain is correct."""

    def test_base_inherits_from_exception(self) -> None:
        assert issubclass(HealthCheckError, Exception)

    def test_unreachable_inherits_from_base(self) -> None:
        assert issubclass(ServiceUnreachableError, HealthCheckError)

    def test_timeout_inherits_from_base(self) -> None:
        assert issubclass(HealthCheckTimeoutError, HealthCheckError)

    def test_catch_specific_with_base_class(self) -> None:
        """Both specific errors must be catchable as HealthCheckError."""
        with pytest.raises(HealthCheckError):
            raise ServiceUnreachableError("svc-a", "Connection refused")

        with pytest.raises(HealthCheckError):
            raise HealthCheckTimeoutError("svc-b", 5.0)


class TestServiceUnreachableError:
    """ServiceUnreachableError: the service could not be contacted at all."""

    def test_preserves_service_name(self) -> None:
        err = ServiceUnreachableError("payment-service", "Connection refused")
        assert err.service_name == "payment-service"

    def test_preserves_reason(self) -> None:
        err = ServiceUnreachableError("payment-service", "Connection refused")
        assert err.reason == "Connection refused"

    def test_str_contains_service_name(self) -> None:
        err = ServiceUnreachableError("payment-service", "Connection refused")
        assert "payment-service" in str(err)

    def test_str_contains_reason(self) -> None:
        err = ServiceUnreachableError("payment-service", "Connection refused")
        assert "Connection refused" in str(err)


class TestHealthCheckTimeoutError:
    """HealthCheckTimeoutError: the service did not respond in time."""

    def test_preserves_service_name(self) -> None:
        err = HealthCheckTimeoutError("slow-service", 3.0)
        assert err.service_name == "slow-service"

    def test_preserves_timeout_seconds(self) -> None:
        err = HealthCheckTimeoutError("slow-service", 3.0)
        assert err.timeout_seconds == 3.0

    def test_str_contains_service_name(self) -> None:
        err = HealthCheckTimeoutError("slow-service", 3.0)
        assert "slow-service" in str(err)
```

### `tests/test_retry.py`

```python
"""
Tests for the retry decorator.
===============================
Your decorator must:
  - Accept max_attempts and delay as configuration
  - Work on async functions (this is the tricky part)
  - Only retry on HealthCheckError (and its subclasses)
  - Let all other exceptions propagate immediately
  - Preserve the decorated function's return value
"""

import time

import pytest

from src.exceptions import HealthCheckError, ServiceUnreachableError
from src.retry import retry


class TestRetryHappyPath:
    """Cases where retry either isn't needed or recovers successfully."""

    @pytest.mark.asyncio
    async def test_no_retry_needed(self) -> None:
        """Functions that succeed on the first call should NOT be retried."""
        call_count = 0

        @retry(max_attempts=3, delay=0.1)
        async def works_fine() -> str:
            nonlocal call_count
            call_count += 1
            return "ok"

        result = await works_fine()
        assert result == "ok"
        assert call_count == 1

    @pytest.mark.asyncio
    async def test_recovers_from_transient_failure(self) -> None:
        """Fails twice, succeeds on the third attempt."""
        call_count = 0

        @retry(max_attempts=3, delay=0.1)
        async def flaky() -> str:
            nonlocal call_count
            call_count += 1
            if call_count < 3:
                raise ServiceUnreachableError("test-svc", "temporary")
            return "recovered"

        result = await flaky()
        assert result == "recovered"
        assert call_count == 3

    @pytest.mark.asyncio
    async def test_preserves_return_value(self) -> None:
        @retry(max_attempts=2, delay=0.1)
        async def returns_dict() -> dict[str, int]:
            return {"count": 42}

        result = await returns_dict()
        assert result == {"count": 42}


class TestRetryExhaustion:
    """Cases where all attempts fail."""

    @pytest.mark.asyncio
    async def test_raises_after_all_attempts_exhausted(self) -> None:
        """When every attempt fails, the LAST exception must propagate."""
        call_count = 0

        @retry(max_attempts=3, delay=0.1)
        async def always_fails() -> str:
            nonlocal call_count
            call_count += 1
            raise ServiceUnreachableError("test-svc", f"attempt {call_count}")

        with pytest.raises(ServiceUnreachableError):
            await always_fails()
        assert call_count == 3

    @pytest.mark.asyncio
    async def test_max_attempts_one_means_no_retry(self) -> None:
        """max_attempts=1 means try once and give up — zero retries."""
        call_count = 0

        @retry(max_attempts=1, delay=0.1)
        async def one_shot() -> str:
            nonlocal call_count
            call_count += 1
            raise ServiceUnreachableError("test-svc", "fail")

        with pytest.raises(ServiceUnreachableError):
            await one_shot()
        assert call_count == 1

    @pytest.mark.asyncio
    async def test_respects_custom_max_attempts(self) -> None:
        call_count = 0

        @retry(max_attempts=5, delay=0.05)
        async def stubborn() -> str:
            nonlocal call_count
            call_count += 1
            raise ServiceUnreachableError("test-svc", "still failing")

        with pytest.raises(ServiceUnreachableError):
            await stubborn()
        assert call_count == 5


class TestRetryTiming:
    """Verify that the delay between retries actually happens."""

    @pytest.mark.asyncio
    async def test_delay_between_retries(self) -> None:
        """
        3 attempts with 0.2s delay → 2 gaps → at least ~0.4s total.
        If this finishes instantly, you're not actually delaying.
        """

        @retry(max_attempts=3, delay=0.2)
        async def always_fails() -> str:
            raise ServiceUnreachableError("test-svc", "fail")

        start = time.monotonic()
        with pytest.raises(ServiceUnreachableError):
            await always_fails()
        elapsed = time.monotonic() - start

        assert elapsed >= 0.35, (
            f"Took only {elapsed:.2f}s — are you actually delaying between retries?"
        )


class TestRetrySelectivity:
    """The retry decorator must only catch HealthCheckError and subclasses."""

    @pytest.mark.asyncio
    async def test_does_not_retry_non_health_check_errors(self) -> None:
        """
        ValueError is NOT a HealthCheckError.
        It should propagate immediately without retry.
        """
        call_count = 0

        @retry(max_attempts=3, delay=0.1)
        async def bad_code() -> str:
            nonlocal call_count
            call_count += 1
            raise ValueError("this is a bug, not a health check failure")

        with pytest.raises(ValueError):
            await bad_code()
        assert call_count == 1, (
            "Should NOT retry on non-HealthCheckError exceptions"
        )

    @pytest.mark.asyncio
    async def test_retries_any_health_check_error_subclass(self) -> None:
        """The base HealthCheckError should also trigger retries."""
        call_count = 0

        @retry(max_attempts=2, delay=0.1)
        async def raises_base() -> str:
            nonlocal call_count
            call_count += 1
            raise HealthCheckError("generic failure")

        with pytest.raises(HealthCheckError):
            await raises_base()
        assert call_count == 2
```

### `tests/test_checker.py`

```python
"""
Tests for async health-checking logic.
=======================================
Your checker.py must export two async functions:

    check_service(config: ServiceConfig) -> HealthResult
        Check a single service. Returns a HealthResult for successful
        HTTP responses. Raises a HealthCheckError subclass for
        connection failures and timeouts.

    check_all_services(configs: list[ServiceConfig]) -> list[HealthResult]
        Check ALL services concurrently. Always returns a list of
        HealthResult — never crashes, even when individual services fail.
"""

import time

import pytest

from src.checker import check_all_services, check_service
from src.exceptions import HealthCheckTimeoutError, ServiceUnreachableError
from src.models import HealthResult, ServiceConfig


# ---------------------------------------------------------------------------
# Fixtures — reusable test configurations
# ---------------------------------------------------------------------------

@pytest.fixture
def healthy_config() -> ServiceConfig:
    return ServiceConfig(
        name="test-healthy",
        url="http://mock:8001/healthy",
        expected_status=200,
        timeout=5.0,
    )


@pytest.fixture
def unhealthy_config() -> ServiceConfig:
    """Service responds with 500. It IS reachable — just sick."""
    return ServiceConfig(
        name="test-unhealthy",
        url="http://mock:8002/unhealthy",
        expected_status=200,
        timeout=5.0,
    )


@pytest.fixture
def unreachable_config() -> ServiceConfig:
    """Service cannot be contacted at all."""
    return ServiceConfig(
        name="test-unreachable",
        url="http://mock:8003/unreachable",
        expected_status=200,
        timeout=5.0,
    )


@pytest.fixture
def slow_config() -> ServiceConfig:
    """Service exists but takes way too long to respond."""
    return ServiceConfig(
        name="test-slow",
        url="http://mock:8004/slow",
        expected_status=200,
        timeout=1.0,  # <-- short timeout to trigger HealthCheckTimeoutError
    )


# ---------------------------------------------------------------------------
# Tests for check_service (single service)
# ---------------------------------------------------------------------------

class TestCheckService:
    """
    check_service checks ONE service and either returns a HealthResult
    or raises a domain-specific exception.
    """

    @pytest.mark.asyncio
    async def test_healthy_service_returns_healthy_result(
        self, healthy_config: ServiceConfig
    ) -> None:
        result = await check_service(healthy_config)
        assert isinstance(result, HealthResult)
        assert result.is_healthy is True
        assert result.service_name == healthy_config.name
        assert result.status_code == 200
        assert result.error is None

    @pytest.mark.asyncio
    async def test_healthy_service_measures_response_time(
        self, healthy_config: ServiceConfig
    ) -> None:
        result = await check_service(healthy_config)
        assert result.response_time_ms is not None
        assert result.response_time_ms > 0

    @pytest.mark.asyncio
    async def test_unhealthy_service_returns_unhealthy_result(
        self, unhealthy_config: ServiceConfig
    ) -> None:
        """A 500 response is still a response — return a result, don't crash."""
        result = await check_service(unhealthy_config)
        assert isinstance(result, HealthResult)
        assert result.is_healthy is False
        assert result.status_code == 500

    @pytest.mark.asyncio
    async def test_unreachable_service_raises_domain_error(
        self, unreachable_config: ServiceConfig
    ) -> None:
        """No response at all → raise ServiceUnreachableError."""
        with pytest.raises(ServiceUnreachableError) as exc_info:
            await check_service(unreachable_config)
        assert exc_info.value.service_name == unreachable_config.name

    @pytest.mark.asyncio
    async def test_slow_service_raises_timeout_error(
        self, slow_config: ServiceConfig
    ) -> None:
        """Service takes too long → raise HealthCheckTimeoutError."""
        with pytest.raises(HealthCheckTimeoutError) as exc_info:
            await check_service(slow_config)
        assert exc_info.value.service_name == slow_config.name


# ---------------------------------------------------------------------------
# Tests for check_all_services (batch, concurrent)
# ---------------------------------------------------------------------------

class TestCheckAllServices:
    """
    check_all_services checks MANY services concurrently.
    It must NEVER raise — failed checks become HealthResult(is_healthy=False).
    """

    @pytest.mark.asyncio
    async def test_returns_one_result_per_service(self) -> None:
        configs = [
            ServiceConfig(
                name=f"svc-{i}",
                url=f"http://mock:800{i}/healthy",
                expected_status=200,
                timeout=5.0,
            )
            for i in range(3)
        ]
        results = await check_all_services(configs)
        assert len(results) == 3
        assert all(isinstance(r, HealthResult) for r in results)

    @pytest.mark.asyncio
    async def test_runs_concurrently_not_sequentially(self) -> None:
        """
        5 healthy services × ~0.5s each.
        Sequential = ~2.5s.  Concurrent = ~0.7s.
        The 2s ceiling gives plenty of margin while still catching sequential code.
        """
        configs = [
            ServiceConfig(
                name=f"svc-{i}",
                url=f"http://mock:800{i}/healthy",
                expected_status=200,
                timeout=5.0,
            )
            for i in range(5)
        ]

        start = time.monotonic()
        results = await check_all_services(configs)
        elapsed = time.monotonic() - start

        assert len(results) == 5
        assert elapsed < 2.0, (
            f"Took {elapsed:.1f}s for 5 services — "
            f"are you awaiting each check inside a for-loop?"
        )

    @pytest.mark.asyncio
    async def test_mixed_services_never_crashes(self) -> None:
        """
        A mix of healthy, unhealthy, and unreachable services.
        check_all_services must return a result for EVERY config —
        even the ones that fail internally.
        """
        configs = [
            ServiceConfig(
                name="healthy-one",
                url="http://mock:8001/healthy",
                expected_status=200,
                timeout=5.0,
            ),
            ServiceConfig(
                name="down-one",
                url="http://mock:8002/unreachable",
                expected_status=200,
                timeout=5.0,
            ),
            ServiceConfig(
                name="sick-one",
                url="http://mock:8003/unhealthy",
                expected_status=200,
                timeout=5.0,
            ),
        ]

        results = await check_all_services(configs)
        assert len(results) == 3, "Must return a result for every service"

        by_name = {r.service_name: r for r in results}
        assert by_name["healthy-one"].is_healthy is True
        assert by_name["down-one"].is_healthy is False
        assert by_name["sick-one"].is_healthy is False

    @pytest.mark.asyncio
    async def test_failed_results_include_error_message(self) -> None:
        configs = [
            ServiceConfig(
                name="gone",
                url="http://mock:8001/unreachable",
                expected_status=200,
                timeout=5.0,
            ),
        ]
        results = await check_all_services(configs)
        assert len(results) == 1
        assert results[0].is_healthy is False
        assert results[0].error is not None
        assert len(results[0].error) > 0, (
            "Error message must be meaningful, not an empty string"
        )

    @pytest.mark.asyncio
    async def test_empty_config_list(self) -> None:
        """No services to check → return an empty list, don't crash."""
        results = await check_all_services([])
        assert results == []
```

---

# Implementation Guide

Work through these files in order. Each step builds on the last, and the corresponding tests become runnable as you go. Do **not** try to implement everything at once.

---

## Step 1 → `models.py` · Data Models (Estimated: 15–30 min)

**Lectures used:** L1 (type hints, `Optional`), L2 (dataclasses)

This file defines the vocabulary of your entire project. Every other file imports from here.

You are building two dataclasses that represent the **input** and **output** of a health check:

**`ServiceConfig`** is the input. It describes one service you want to check. It holds everything the checker needs to know before making a request: what the service is called, where to reach it, what HTTP status code means "healthy," and how long to wait before giving up. All four fields are always present — nothing is optional.

**`HealthResult`** is the output. It describes what happened when you checked one service. It holds the service's name, whether it's healthy, how long the response took, what status code came back, and an error message if something went wrong. Here's the key design challenge: not every field makes sense in every situation. If the service was unreachable, there is no response time and no status code. If the service responded normally, there is no error message. You need to choose type hints that let certain fields be `None` while others are always present. Look at how the tests construct `HealthResult` with `None` values — that tells you which fields need this treatment.

**Checkpoint:** `pytest tests/test_models.py -v` → all green.

**Commit:** `feat: add ServiceConfig and HealthResult data models`

---

## Step 2 → `exceptions.py` · Exception Hierarchy (Estimated: 15–20 min)

**Lectures used:** L2 (custom exceptions, inheritance, `__init__` / `__str__`)

When a health check fails, your system needs to communicate *how* it failed. Python's built-in exceptions like `ConnectionError` and `TimeoutError` are too generic — they don't carry information about which service failed. Your checker needs domain-specific exceptions that carry that context.

You are building a three-class exception hierarchy:

```
Exception              ← built-in, you don't touch this
└── HealthCheckError           ← your base class for ALL health check failures
    ├── ServiceUnreachableError    ← the service couldn't be contacted at all
    └── HealthCheckTimeoutError    ← the service didn't respond within the time limit
```

**`HealthCheckError`** is the common base. It exists so that other parts of your code (like the retry decorator) can catch *any* health check failure with a single `except HealthCheckError` clause, regardless of the specific failure mode. It inherits from Python's built-in `Exception`.

**`ServiceUnreachableError`** represents a complete connection failure — the service is down, the network is broken, or the address is wrong. This exception must store two pieces of context: the `service_name` (which service failed) and a `reason` (a human-readable explanation like `"Connection refused"`). Both must be accessible as instance attributes, and both must appear when you `print()` or `str()` the exception.

**`HealthCheckTimeoutError`** represents a service that exists but takes too long to respond. It stores the `service_name` and the `timeout_seconds` (how long the caller was willing to wait). Again, both must be instance attributes and both must appear in the string representation.

The reason these are separate classes rather than one generic error is that different failure modes may need different handling strategies in the future. Unreachable might mean "page the on-call engineer." Timeout might mean "try again in 30 seconds." Your hierarchy makes that possible.

**Checkpoint:** `pytest tests/test_exceptions.py -v` → all green.

**Commit:** `feat: add health check exception hierarchy`

---

## Step 3 → `retry.py` · Retry Decorator (Estimated: 30–60 min)

**Lectures used:** L2 (decorators), L3 (async/await, `asyncio.sleep`)

Network calls fail. Services go down for a second and come back. Your system needs to handle these **transient failures** by trying again instead of giving up immediately. You are building a decorator called `retry` that adds this "try again" behavior to any async function — without modifying the function itself.

Here is what `retry` does, described as a black box:

```python
@retry(max_attempts=3, delay=0.5)
async def some_async_function():
    ...
```

When `some_async_function()` is called, the decorator intercepts the call and runs this logic:

1. **Attempt** the original function.
2. If it **succeeds**, return the result immediately. Done. No more attempts.
3. If it raises a **`HealthCheckError`** (or any subclass), that's a transient failure. **Wait** for `delay` seconds (using the async-friendly sleep, not `time.sleep`), then go back to step 1.
4. If it raises **any other exception** (like `ValueError`, `TypeError`), that's a bug — not a network problem. Let it propagate immediately. Do **not** retry.
5. If you've used all `max_attempts` attempts and the function still hasn't succeeded, **raise the last exception** you caught. Don't swallow it.

So `max_attempts=3` means: try once, and if that fails, retry up to 2 more times — 3 total attempts, with a delay between each retry.

The tricky part: this decorator wraps an **async** function. That means:

- The inner wrapper function must itself be declared `async def`, because it needs to `await` the decorated function.
- The delay between retries must use `asyncio.sleep()`, not `time.sleep()`. (Remember from Lecture 3: `time.sleep()` blocks the entire event loop.)
- The decorator still follows the same structure as any decorator from Lecture 2 — an outer function that takes configuration (`max_attempts`, `delay`), a middle function that takes the function being decorated, and an inner function (the wrapper) that runs the actual logic. The only difference is that the innermost wrapper is `async def` instead of plain `def`.

Think of it as a loop with at most `max_attempts` iterations. Each iteration tries calling the function. If the call succeeds, you break out and return the value. If it raises the right kind of exception, you sleep and try again. The challenge is getting the structure right: where does the loop go, where does the try/except go, and what happens after the loop ends?

**Checkpoint:** `pytest tests/test_retry.py -v` → all green.

**Commit:** `feat: implement async retry decorator`

---

## Step 4 → `checker.py` · Async Health Checker (Estimated: 60–90 min)

**Lectures used:** L1 (type hints), L2 (context managers, error handling), L3 (async/await, `asyncio.gather`, `asyncio.wait_for`), L4 (project structure)

This is the core file. It ties every concept from the first four lectures together.

You are building two async functions:

**`check_service(config: ServiceConfig) -> HealthResult`** checks a single service. Here is what it does step by step:

1. **Open a simulated HTTP session** using the `SimulatedSession` from `_simulator.py`. This must be done with `async with` so the session is properly cleaned up even if something goes wrong. (This is the async version of the context manager pattern from Lecture 2 — the only syntactic difference is `async with` instead of `with`.)

2. **Make the request with a timeout.** Call `session.request(config.url)`, but don't let it run forever. You need to enforce `config.timeout` seconds as a maximum wait. Look up `asyncio.wait_for` in the standard library reference — it wraps any coroutine with a timeout and raises `asyncio.TimeoutError` if time runs out.

3. **Measure response time.** Record the time before and after the request (use `time.monotonic()`). Convert the elapsed time to milliseconds for the `HealthResult`.

4. **Evaluate the response.** If the simulator returns a response, compare its `status_code` to `config.expected_status`. If they match → `is_healthy=True`. If they don't match (e.g., you expected 200 but got 500) → `is_healthy=False`, with an error message explaining the mismatch. In both cases, **return a `HealthResult`**.

5. **Translate low-level exceptions into domain exceptions.** Two things can go wrong before you get a response:
   - The simulator raises `ConnectionError` (service unreachable) → catch it and raise your `ServiceUnreachableError` instead.
   - `asyncio.wait_for` raises `asyncio.TimeoutError` (service too slow) → catch it and raise your `HealthCheckTimeoutError` instead.

   In both cases, **raise** — don't return a `HealthResult`. The caller decides how to handle the exception.

Notice the design: `check_service` returns a `HealthResult` for successful HTTP responses (even bad ones like 500), but *raises* an exception when it can't get a response at all. This is an intentional design choice that the tests enforce.

**`check_all_services(configs: list[ServiceConfig]) -> list[HealthResult]`** checks many services concurrently and always returns a complete list of results — one per service, no exceptions, no crashes.

Here is what it does:

1. **Launch all checks concurrently.** Do not loop through configs and `await` each one sequentially. Instead, create all the coroutines and run them at the same time using `asyncio.gather()`. This is the whole reason the project exists — turning a 1-minute sequential process into a 2-second concurrent one.

2. **Handle failures gracefully.** Some of those concurrent checks will raise exceptions (unreachable, timeout). This function must catch those exceptions and convert them into `HealthResult(is_healthy=False, error=...)`. The caller of `check_all_services` should never see an exception — they get a clean list of results they can iterate over.

   There are multiple valid approaches here. You could wrap each individual coroutine in a helper that catches exceptions before passing them to `gather`. You could use `asyncio.gather`'s `return_exceptions` parameter and process the results afterward. The tests don't prescribe which approach you use — they only check that the output is correct and that the timing proves concurrency.

3. **Return an empty list for empty input.** If given zero configs, return `[]`. Don't crash.

**Checkpoint:** `pytest tests/test_checker.py -v` → all green.

**Commit:** `feat: implement async health checker`

---

## Step 5 → `main.py` · Entry Point (Estimated: 20–30 min)

**Lectures used:** L3 (`asyncio.run`), L1 (type hints for JSON parsing), L4 (project structure)

This is your integration point — the file you run to see the whole system work end-to-end. There are no pre-written tests for this file, but it should work correctly when you run `python -m src.main`.

You are building a script that:

1. **Loads `services.json`** from disk using Python's built-in `json` module. The file contains a list of objects, each with `name`, `url`, `expected_status`, and `timeout` fields.

2. **Converts the raw JSON data** into a list of `ServiceConfig` instances. Each dict from the JSON becomes one `ServiceConfig`.

3. **Runs `check_all_services`** with those configs. Since this is an async function and `main.py` is the synchronous entry point, you'll need `asyncio.run()` to bridge from sync to async.

4. **Prints a readable report** of the results. Format is up to you, but it should be clear at a glance which services are healthy and which aren't.

**Commit:** `feat: add main entry point with JSON config loading`

---

## Step 6 → Type Check and Final Cleanup

```bash
mypy src/
```

Fix any type errors. Review your code for dead imports, unused variables, and unclear naming.

**Commit:** `fix: resolve mypy strict mode errors`

# Lecture → Task Quick Reference

| Lecture | Concepts | Where You'll Use Them |
| :--- | :--- | :--- |
| **L1: Type Hints** | `Optional`, `list[X]`, `dict[K, V]`, type aliases, `mypy --strict` | Every file. `HealthResult` fields that can be `None`. Function signatures everywhere. The final `mypy` pass. |
| **L2: Python Patterns** | Dataclasses, context managers, decorators, custom exceptions, try/except/finally | `models.py` (dataclasses), `exceptions.py` (custom exceptions), `retry.py` (decorator), `checker.py` (context manager + error handling) |
| **L3: Async** | `async/await`, `asyncio.gather`, event loop model, blocking vs non-blocking | `checker.py` (core async logic), `retry.py` (async-aware decorator) |
| **L4: Workflow** | Git, branches, conventional commits, venv, pyproject.toml | Project setup, every commit message, your PR submission |

---

# Useful Standard Library Reference

These are tools from Python's standard library that you **will need** for this project. They are all natural extensions of what you learned in lectures. We list them here without explanation — look them up, read the signature, try them in a REPL.

| Tool | Module | One-Line Hint |
| :--- | :--- | :--- |
| `asyncio.wait_for(coro, timeout=...)` | `asyncio` | Wraps a coroutine with a timeout. What does it raise when time runs out? |
| `asyncio.gather(*coros)` | `asyncio` | Run multiple coroutines concurrently and collect all results. |
| `asyncio.sleep(seconds)` | `asyncio` | Non-blocking sleep. NOT the same as `time.sleep()`. |
| `asyncio.run(coro)` | `asyncio` | Run an async function from synchronous code (for `main.py`). |
| `time.monotonic()` | `time` | High-precision clock for measuring elapsed time. |
| `json.load(file)` | `json` | Parse JSON from a file object into Python dicts/lists. |
| `dataclasses.dataclass` | `dataclasses` | The `@dataclass` decorator. You saw this in Lecture 2. |

### One Concept Not Explicitly in Lectures

**`async with`** — Python supports async context managers. They work exactly like regular `with` blocks (Lecture 2), but `__aenter__` and `__aexit__` are `async`. You use them with the `async with` syntax:

```python
async with SomeAsyncResource() as resource:
    await resource.do_something()
# resource is cleaned up here, even if an exception occurred
```

This is the natural combination of Lecture 2 (context managers) and Lecture 3 (async). The simulator's `SimulatedSession` requires this pattern.

---

## ⚠️ Watch Out (Read Before You Start)

These are real traps that trip up students every time. I'm naming them, not solving them.

**1. The Silent Sequential.** If you `await` each check one after another inside a `for` loop, your code works but is sequential. The timing test will catch you. Think about what `asyncio.gather()` does differently.

**2. The Coroutine That Never Ran.** If you forget to `await` a coroutine, Python gives you a warning — but your program doesn't crash. It just silently skips work. Check your terminal output carefully for `RuntimeWarning: coroutine '...' was never awaited`.

**3. The Bare Except.** Writing `except:` or `except Exception:` will make some tests pass but violates B4. The retry decorator must only catch `HealthCheckError` — not all exceptions. Think about why: if your code has a genuine bug (e.g., `TypeError`), you don't want to silently retry it three times and hide the error.

**4. Blocking in Async Land.** `time.sleep(1)` inside an `async def` freezes your **entire event loop** for 1 second — no other task can run. There's an async equivalent. Look at how `_simulator.py` handles delays.

**5. The Async Decorator Trap.** Your retry decorator wraps an `async` function. That means the wrapper function must also be `async`, and it must `await` the decorated function. If you've only written decorators for sync functions before, you'll need to adjust your mental model.

**6. Timeout ≠ Slow Response.** An unhealthy service (status 500) responds quickly. A slow service doesn't respond within your timeout. These are different failure modes handled differently. Read the tests carefully to see which case raises an exception and which returns a `HealthResult`.

**7. The Missing `async with`.** If you call `SimulatedSession().request(url)` without using `async with`, you'll get a `RuntimeError`. The simulator enforces this deliberately because real HTTP libraries have the same requirement. This is your signal to use a context manager.

---

## Submission Instructions

### 1. Verify Everything Passes

```bash
# All tests green
pytest tests/ -v

# Zero type errors
mypy src/

# Both commands must succeed before you submit
```

### 2. Review Your Commit History

Your git log should tell the story of how you built the project. Aim for **at least 5 meaningful commits** using conventional commit format:

```
feat: add ServiceConfig and HealthResult data models
feat: add health check exception hierarchy
feat: implement async retry decorator
feat: implement async health checker
feat: add main entry point with JSON config loading
fix: resolve mypy strict mode errors
```

**Bad commits:** `fix stuff`, `wip`, `asdfasdf`, `final version`, `final final version`

### 3. Push and Open a Pull Request

```bash
git push origin feat/health-checker
```

Open a Pull Request from `feat/health-checker` → `main` on the course repository. Your PR description should include:

- A one-sentence summary of what the project does
- Instructions to run it (`python -m src.main`)
- Confirmation that `pytest` and `mypy` both pass

---

## Rubric

Visible from Day 1. No surprises.

| Category | Weight | Criteria |
| :--- | :--- | :--- |
| **Type Correctness** | 20% | `mypy --strict` passes with **zero** errors on `src/`. Binary: pass or fail. |
| **Async Correctness** | 25% | `test_checker.py` passes in full, including the timing assertion. Concurrency must be real. |
| **Pattern Usage** | 25% | Dataclasses for models. `async with` for session management. Retry implemented as a decorator. Exception hierarchy with a shared base class. |
| **Error Handling** | 15% | Domain-specific exceptions used internally. `check_all_services` never crashes. Error messages are meaningful. No bare `except`. |
| **Git Workflow** | 10% | Submitted via PR. ≥5 meaningful commits. Conventional commit format. Feature branch used. |
| **Code Clarity** | 5% | Another human can read your code without the spec. Good naming, no dead code, reasonable structure. |

---

## FAQ

**Q: The tests won't even run — I get `ModuleNotFoundError`.**
Did you run `pip install -e ".[dev]"` from inside your activated virtual environment? This installs the project so `from src.models import ...` works. Check with `which python` (macOS/Linux) or `where python` (Windows) — it should point to your `.venv` directory.

**Q: All tests fail immediately with `ImportError: cannot import name 'ServiceConfig'`.**
That means `models.py` is empty or doesn't define `ServiceConfig` yet. This is expected when you first start. Implement `models.py` first — other test files also import from `models`.

**Q: My async tests hang forever and never finish.**
You probably have a `time.sleep()` call inside an `async def`. That blocks the entire event loop. Replace it with the async equivalent. Also check that you're not awaiting the slow simulator without a timeout.

**Q: `mypy` complains about the test files.**
It shouldn't — `pyproject.toml` is configured to only check `src/`. If you're running `mypy .` instead of `mypy src/`, you'll see test file warnings. Run `mypy src/` specifically.

**Q: How do I know my concurrency actually works?**
The timing test in `test_checker.py` is your proof. But you can also add a quick sanity check: put a `print(f"Starting {config.name}")` at the top of `check_service` and `print(f"Finished {config.name}")` at the bottom. If your output interleaves (starts appear together, then finishes appear together), you're concurrent. If it alternates start-finish-start-finish, you're sequential.

**Q: Can I add my own tests?**
Yes. Create new test files or add tests to existing ones. Do **not** modify or delete the pre-written tests.

**Q: Can I add helper functions or extra files?**
Yes. You can add private helper functions, utility modules, or anything else inside `src/`. The pre-written tests only import the names specified — everything else is your design space.

**Q: What's the difference between `check_service` raising an exception and `check_all_services` returning `is_healthy=False`?**
`check_service` is the low-level function. It raises `ServiceUnreachableError` or `HealthCheckTimeoutError` when things go wrong — this is how it *signals* failure. `check_all_services` is the high-level function. It *catches* those exceptions and converts them into `HealthResult(is_healthy=False, error=...)`. This pattern — exceptions internally, structured results externally — is common in production backend services.