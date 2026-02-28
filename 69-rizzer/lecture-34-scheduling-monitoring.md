# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCENARIO FIRST, TOOL SECOND                                    │
│  ───────────────────────────                                    │
│  Students must feel the operational pain of unmonitored,        │
│  unscheduled background work before they see the fix.           │
│  We'll let things break on screen.                              │
│                                                                 │
│  OPERATIONS MINDSET SHIFT                                       │
│  ────────────────────────                                       │
│  This is not about writing code. It is about running systems.   │
│  The question is no longer "does it work?"                      │
│  It is "does it work at 3 AM when nobody is watching?"          │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  A factory production floor analogy carries through every       │
│  concept. Schedule → Protect → Observe → React.                 │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Celery tasks (Lecture 2) → now we schedule them                │
│  Error handling / retries (Lecture 2) → now we handle the       │
│    cases when retries aren't enough                             │
│  Logging (Week 3) → now we build alerting on top of it          │
│  Redis (Week 10) → Beat uses Redis for schedule state           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                  SCHEDULING & MONITORING                        │
│                     (3-4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (25 min)                                   │
│  ├─ 1.1 The 3 AM Question (Demonstration)                       │
│  ├─ 1.2 The while-True Anti-Pattern                             │
│  └─ 1.3 The Factory Floor Analogy                               │
│                                                                 │
│  PART 2: CELERY BEAT — THE SCHEDULER (50 min)                   │
│  ├─ 2.1 What is Celery Beat?                                    │
│  ├─ 2.2 Interval Schedules (timedelta)                          │
│  ├─ 2.3 Crontab Schedules (crontab)                             │
│  ├─ 2.4 Configuring beat_schedule                               │
│  ├─ 2.5 Running Beat                                            │
│  └─ 2.6 The Single-Beat Rule                                    │
│                                                                 │
│  PART 3: TASK TIMEOUTS — SAFETY SWITCHES (40 min)               │
│  ├─ 3.1 Why Tasks Need Timeouts                                 │
│  ├─ 3.2 soft_time_limit (The Warning Bell)                      │
│  ├─ 3.3 time_limit (The Emergency Stop)                         │
│  ├─ 3.4 Setting Timeouts (Per-Task and Global)                  │
│  └─ 3.5 Graceful Cleanup on Timeout                             │
│                                                                 │
│  PART 4: DEAD LETTER HANDLING (30 min)                          │
│  ├─ 4.1 What is a Poison Message?                               │
│  ├─ 4.2 When Retries Aren't Enough                              │
│  ├─ 4.3 The on_failure Hook                                     │
│  └─ 4.4 Building a Dead Letter Store                            │
│                                                                 │
│  PART 5: MONITORING WITH FLOWER (30 min)                        │
│  ├─ 5.1 Why Monitoring Matters                                  │
│  ├─ 5.2 Flower Setup                                            │
│  ├─ 5.3 Reading the Dashboard                                   │
│  └─ 5.4 Worker Inspection from Code                             │
│                                                                 │
│  PART 6: ALERTING ON FAILURES (25 min)                          │
│  ├─ 6.1 The Dashboard Nobody Watches                            │
│  ├─ 6.2 Celery Signals                                          │
│  ├─ 6.3 Health Check Endpoints                                  │
│  └─ 6.4 Common Mistakes                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The 3 AM Question

**Start with a scenario. Not a concept — a disaster.**

> "You shipped your background task system from Lecture 2. Tasks trigger when users do things — great. But your PM just posted three new requirements."

```
┌─────────────────────────────────────────────────────────────────┐
│                  NEW REQUIREMENTS (Monday Morning)              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Every hour: delete expired JWT refresh tokens from Redis    │
│     (Remember Week 9 auth? Tokens pile up. Redis isn't free.)   │
│                                                                 │
│  2. Every midnight: generate a daily usage report and store it  │
│     in the database for the admin dashboard                     │
│                                                                 │
│  3. Every 5 minutes: check if the external APIs we depend on    │
│     (from Week 8) are still responding                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "None of these are triggered by a user action. There is no HTTP request. There is no button. These need to happen **on a clock**, automatically, forever. How would you build this with what you know right now?"

**Pause. Let them think.**

Most students will arrive at the same answer. Let them say it.

---

## 1.2 The while-True Anti-Pattern

**Show the answer they're thinking of:**

```python
# naive_scheduler.py — "It works on my laptop"
import time
from tasks import cleanup_tokens, generate_report, check_api_health

def run_scheduler():
    last_cleanup = 0
    last_report = 0
    last_health = 0

    while True:
        now = time.time()

        if now - last_health >= 300:          # Every 5 minutes
            check_api_health.delay()
            last_health = now

        if now - last_cleanup >= 3600:        # Every hour
            cleanup_tokens.delay()
            last_cleanup = now

        if now - last_report >= 86400:        # Every 24 hours
            generate_report.delay()
            last_report = now

        time.sleep(10)  # Check every 10 seconds

if __name__ == "__main__":
    run_scheduler()
```

**Now ask the class:**

> "This works. Run the script, tasks get scheduled. Ship it. What could go wrong?"

**Let them brainstorm. Then reveal the answer, one disaster at a time:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT GOES WRONG WITH while-True                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❶ THE SCRIPT CRASHES                                           │
│     An unhandled exception at 2:47 AM. The process dies.        │
│     No more scheduled tasks. Nobody notices until Monday.       │
│     3 days of missed token cleanup. Redis is full.              │
│                                                                 │
│  ❷ THE SERVER REBOOTS                                           │
│     Kernel update, hardware issue, cloud provider maintenance.  │
│     Your script was running in a terminal tab.                  │
│     It did not come back.                                       │
│                                                                 │
│  ❸ TIME DRIFT                                                   │
│     time.sleep(10) doesn't sleep for exactly 10 seconds.        │
│     Over weeks, your "every midnight" report drifts.            │
│     Now it runs at 12:04 AM. Then 12:11 AM. Then...            │
│                                                                 │
│  ❹ YOU DEPLOY A NEW VERSION                                     │
│     You restart the app. The script restarts.                   │
│     All the last_cleanup / last_report timestamps reset to 0.   │
│     Every scheduled task fires immediately. At once.            │
│                                                                 │
│  ❺ A TASK HANGS FOREVER                                         │
│     generate_report calls an external API that never responds.  │
│     The task sits there. Consuming a worker. Holding a DB       │
│     connection. Forever. You have no idea it happened.          │
│                                                                 │
│  ❻ TASKS FAIL SILENTLY                                          │
│     cleanup_tokens has a bug. It raises an exception.           │
│     Celery retries 3 times, gives up.                           │
│     Nobody notices. Expired tokens pile up for weeks.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The real question:**

> "This lecture answers one thing: **How do you build background work that runs itself, protects itself, and tells you when it's broken?** That means four capabilities: scheduling, timeouts, dead letter handling, and monitoring with alerts."

```
┌─────────────────────────────────────────────────────────────────┐
│               THE FOUR CAPABILITIES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│        SCHEDULE          →   Celery Beat                        │
│        "Run this task every hour"                               │
│                                                                 │
│        PROTECT           →   Task Timeouts                      │
│        "Kill it if it hangs longer than 2 minutes"              │
│                                                                 │
│        ESCALATE          →   Dead Letter Handling               │
│        "If it fails 3 times, stop retrying and flag it"         │
│                                                                 │
│        OBSERVE + REACT   →   Flower + Alerting                  │
│        "Show me what's running, and page me when it breaks"     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 The Factory Floor Analogy

**This analogy will carry through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE FACTORY FLOOR ANALOGY                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your background task system is a FACTORY.                      │
│                                                                 │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │  PRODUCTION │    │   MACHINES   │    │   CONTROL    │       │
│  │  SCHEDULE   │───▶│  (Workers)   │───▶│     ROOM     │       │
│  │  (Beat)     │    │              │    │   (Flower)   │       │
│  └─────────────┘    └──────┬───────┘    └──────────────┘       │
│                            │                                    │
│                     ┌──────┴───────┐                            │
│                     │              │                            │
│               ┌─────┴─────┐  ┌────┴──────┐                     │
│               │  SAFETY   │  │  REJECT   │                     │
│               │  SHUTOFF  │  │    BIN    │                     │
│               │ (Timeout) │  │(Dead Ltrs)│                     │
│               └───────────┘  └───────────┘                     │
│                                                                 │
│                                                                 │
│  Factory Concept           │  Celery Equivalent                │
│  ──────────────────────────│───────────────────────────         │
│  Production schedule board │  beat_schedule configuration      │
│  Shift manager             │  Celery Beat process              │
│  Machines on the floor     │  Celery workers                   │
│  Products being made       │  Tasks being executed             │
│  Safety auto-shutoff       │  soft_time_limit                  │
│  Emergency power cut       │  time_limit (hard kill)           │
│  Defective → reject bin    │  Dead letter handling             │
│  Control room monitors     │  Flower dashboard                 │
│  Alarm siren / pager       │  Celery signals + alerts          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "A well-run factory doesn't need the owner standing on the floor 24/7. The schedule runs itself. The machines have safety switches. Defective products get caught and set aside. The control room shows everything. And alarms go off when something is actually wrong. That's what we're building today."

---

# PART 2: CELERY BEAT — THE SCHEDULER

## 2.1 What is Celery Beat?

**Celery Beat is a separate process that sends task messages on a schedule.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     CELERY BEAT                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Beat does NOT execute tasks.                                   │
│  Beat SENDS task messages to the broker at the right time.      │
│  Workers pick them up and execute them — same as always.        │
│                                                                 │
│                                                                 │
│   ┌──────────┐   sends message   ┌─────────┐   picks up        │
│   │  CELERY  │  ──────────────▶  │  REDIS  │  ──────────▶      │
│   │   BEAT   │   at 00:00 UTC    │ (broker)│             ┌─────┐│
│   │(scheduler│   at */5 min      │  queue  │             │WORKR││
│   │ process) │   at */1 hour     │         │             │     ││
│   └──────────┘                   └─────────┘             └─────┘│
│                                                                 │
│                                                                 │
│  This is exactly like .delay() or .apply_async()                │
│  from Lecture 2 — but instead of YOUR CODE sending the          │
│  message, BEAT sends it on a clock.                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key insight:**

> "Beat is the shift manager who walks up to the production schedule board every few seconds, checks the clock, and says 'It's midnight — time to start the reporting job.' Then it walks over to the conveyor belt (Redis queue) and drops a work order on it. The machines (workers) pick it up and do the work. Beat never touches the product itself."

---

## 2.2 Interval Schedules (timedelta)

**The simplest schedule: "do this every N seconds/minutes/hours."**

```python
from datetime import timedelta

# These are the tasks you already have from Lecture 2.
# Nothing changes about how tasks are DEFINED.
# Only HOW THEY GET TRIGGERED is new.

# "Run check_api_health every 5 minutes"
# schedule = timedelta(minutes=5)

# "Run cleanup_tokens every 1 hour"
# schedule = timedelta(hours=1)

# "Run sync_external_data every 30 seconds"
# schedule = timedelta(seconds=30)
```

**How intervals work internally:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  INTERVAL SCHEDULE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  schedule = timedelta(minutes=5)                                │
│                                                                 │
│  Time:  00:00   00:05   00:10   00:15   00:20                   │
│           │       │       │       │       │                     │
│    Beat:  📤      📤      📤      📤      📤                     │
│           │       │       │       │       │                     │
│           └───────┴───────┴───────┴───────┘                     │
│              5 min   5 min   5 min   5 min                      │
│                                                                 │
│  📤 = Beat sends a task message to the broker                   │
│                                                                 │
│  Interval starts from when Beat STARTS, not from when           │
│  the task FINISHES. If the task takes 3 minutes and the         │
│  interval is 5 minutes, the next send is still 5 minutes        │
│  after the LAST SEND, not 5 minutes after completion.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When to use intervals:**

> "Intervals are for tasks where the exact wall-clock time doesn't matter. 'Every 5 minutes' — you don't care if it's 12:00 or 12:02. You just want roughly 5-minute gaps."

---

## 2.3 Crontab Schedules (crontab)

**For when exact wall-clock time matters: "every day at midnight," "every Monday at 9 AM."**

```python
from celery.schedules import crontab
```

**Crontab is modeled after the UNIX cron format, but with named keyword arguments instead of the cryptic five-field string:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   CRONTAB FIELDS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  UNIX cron (for reference — you won't type this in Celery):     │
│                                                                 │
│    ┌────── minute (0-59)                                        │
│    │ ┌──── hour (0-23)                                          │
│    │ │ ┌── day of month (1-31)                                  │
│    │ │ │ ┌─ month (1-12)                                        │
│    │ │ │ │ ┌─ day of week (0=Mon ... 6=Sun)                     │
│    │ │ │ │ │                                                    │
│    * * * * *   ← UNIX cron syntax                               │
│                                                                 │
│                                                                 │
│  Celery crontab (what you actually write):                      │
│                                                                 │
│    crontab(                                                     │
│        minute=0,             # default: '*' (every minute)      │
│        hour=0,               # default: '*' (every hour)        │
│        day_of_week='*',      # default: '*' (every day)         │
│        day_of_month='*',     # default: '*' (every day)         │
│        month_of_year='*',    # default: '*' (every month)       │
│    )                                                            │
│                                                                 │
│  The '*' means "every" — same as UNIX cron.                     │
│  Any field you DON'T specify defaults to '*'.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Common schedule patterns — read these carefully:**

```python
from celery.schedules import crontab

# ── EVERY MINUTE ──────────────────────────────────────────
crontab()
# All fields default to '*' → every minute of every hour
# of every day. Almost never what you want.

# ── EVERY HOUR, ON THE HOUR ───────────────────────────────
crontab(minute=0)
# minute=0 → only at minute 0
# hour='*' → every hour
# Result: 00:00, 01:00, 02:00, ... 23:00

# ── EVERY DAY AT MIDNIGHT ─────────────────────────────────
crontab(minute=0, hour=0)
# minute=0, hour=0 → only at 00:00
# day_of_week='*' → every day
# Result: 00:00 daily

# ── EVERY 15 MINUTES ──────────────────────────────────────
crontab(minute='*/15')
# '*/15' means "every 15th minute"
# Result: :00, :15, :30, :45 of every hour

# ── EVERY MONDAY AT 9:00 AM ───────────────────────────────
crontab(minute=0, hour=9, day_of_week=1)
# day_of_week: 0=Monday, 1=Tuesday, ... 6=Sunday
# Result: Tuesday 09:00 every week
#
# ⚠️  WAIT. Read that again. Celery uses 0=Monday.
#     UNIX cron uses 0=Sunday. This is a common trap.

# ── WEEKDAYS AT 8:30 AM ───────────────────────────────────
crontab(minute=30, hour=8, day_of_week='mon-fri')
# You can use names: 'mon', 'tue', 'wed', 'thu', 'fri',
#                     'sat', 'sun'
# Ranges: 'mon-fri'
# Result: 08:30, Monday through Friday

# ── FIRST DAY OF EVERY MONTH AT 6:00 AM ───────────────────
crontab(minute=0, hour=6, day_of_month=1)
# Result: 06:00 on the 1st of every month

# ── EVERY 3 HOURS ─────────────────────────────────────────
crontab(minute=0, hour='*/3')
# Result: 00:00, 03:00, 06:00, 09:00, ... 21:00
```

**Ask the class:**

> "What does `crontab(minute=0, hour='8,12,18')` mean?"

Answer: Runs at 08:00, 12:00, and 18:00 every day. You can pass comma-separated values for specific times.

**Map it visually:**

```
┌─────────────────────────────────────────────────────────────────┐
│             CRONTAB VISUAL: crontab(minute=0, hour='*/3')      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Day timeline:                                                  │
│                                                                 │
│  00  01  02  03  04  05  06  07  08  09  10  11  12             │
│   │           │           │           │           │             │
│   📤          📤          📤          📤          📤             │
│                                                                 │
│  12  13  14  15  16  17  18  19  20  21  22  23  00             │
│   │           │           │           │           │             │
│   📤          📤          📤          📤        (next day)       │
│                                                                 │
│  📤 = task message sent to broker                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Configuring beat_schedule

**Now we connect the schedule to actual tasks.**

You already have tasks defined from Lecture 2. Nothing about the task definition changes. You add a configuration dictionary that tells Beat *which* task to send and *when*.

```python
# celery_app.py — add this to your existing Celery config

from celery import Celery
from celery.schedules import crontab
from datetime import timedelta

celery_app = Celery(
    "worker",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/1",
)

# ── THIS IS THE NEW PART ──────────────────────────────────
celery_app.conf.beat_schedule = {

    # ── Key: a human-readable name (your choice) ──────────
    "cleanup-expired-tokens": {
        "task": "tasks.cleanup_expired_tokens",  # Dotted path to task
        "schedule": crontab(minute=0),           # Every hour, on the hour
        # No args needed for this one
    },

    "generate-daily-report": {
        "task": "tasks.generate_daily_report",
        "schedule": crontab(minute=0, hour=0),   # Every day at midnight
        "args": ("usage",),                      # Positional arguments
    },

    "check-external-apis": {
        "task": "tasks.check_api_health",
        "schedule": timedelta(minutes=5),        # Every 5 minutes
        "kwargs": {"timeout": 10},               # Keyword arguments
    },

    "archive-old-tasks": {
        "task": "tasks.archive_completed_tasks",
        "schedule": crontab(                     # Sundays at 3 AM
            minute=0,
            hour=3,
            day_of_week="sun",
        ),
        "args": (30,),  # Archive tasks older than 30 days
    },
}

# ── TIMEZONE ───────────────────────────────────────────────
# Critical: crontab hours are interpreted in this timezone.
# Always set this explicitly. Never rely on the default.
celery_app.conf.timezone = "UTC"
```

**The structure of each entry:**

```
┌─────────────────────────────────────────────────────────────────┐
│               beat_schedule ENTRY STRUCTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "human-readable-name": {                                       │
│      "task":     "dotted.path.to.task.function",   # REQUIRED   │
│      "schedule": crontab(...) or timedelta(...),   # REQUIRED   │
│      "args":     (arg1, arg2),          # optional, positional  │
│      "kwargs":   {"key": "value"},      # optional, keyword     │
│      "options":  {"queue": "high"},     # optional, task opts   │
│  }                                                              │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ "task" is a STRING — the import path, not a Python  │        │
│  │ reference. This is because Beat sends a MESSAGE     │        │
│  │ with the task name. The WORKER resolves the name    │        │
│  │ to actual code. Beat never imports your tasks.      │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**And the tasks themselves — nothing special:**

```python
# tasks.py — these are normal tasks from Lecture 2
# Beat doesn't change how you DEFINE tasks.

import structlog
from celery_app import celery_app

logger = structlog.get_logger()


@celery_app.task
def cleanup_expired_tokens() -> int:
    """Remove expired refresh tokens from Redis."""
    # Your Redis cleanup logic from Week 10
    count = redis_client.delete_expired_tokens()
    logger.info("token_cleanup_complete", deleted=count)
    return count


@celery_app.task
def generate_daily_report(report_type: str) -> dict:
    """Generate and store a daily usage report."""
    # Query database, aggregate, store result
    report = build_report(report_type)
    store_report(report)
    logger.info("daily_report_generated", type=report_type)
    return {"type": report_type, "rows": report["row_count"]}


@celery_app.task
def check_api_health(timeout: int = 10) -> dict:
    """Ping external APIs and log their status."""
    results = {}
    for api_name, url in EXTERNAL_APIS.items():
        try:
            response = httpx.get(url, timeout=timeout)
            results[api_name] = response.status_code
        except httpx.TimeoutException:
            results[api_name] = "TIMEOUT"
            logger.warning("api_health_timeout", api=api_name)
    return results


@celery_app.task
def archive_completed_tasks(days_old: int) -> int:
    """Move completed tasks older than N days to archive."""
    cutoff = datetime.utcnow() - timedelta(days=days_old)
    count = move_to_archive(completed_before=cutoff)
    logger.info("archive_complete", archived=count, older_than_days=days_old)
    return count
```

> "Notice: the tasks are identical to what you wrote in Lecture 2. They have no awareness that Beat exists. Beat is entirely external — it just sends messages on a timer. The task doesn't know or care whether it was triggered by `task.delay()` in an API endpoint, or by Beat on a schedule, or by you calling it from a Python shell."

---

## 2.5 Running Beat

**Beat is its own process — separate from your workers.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  YOUR RUNNING PROCESSES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE THIS LECTURE (Lecture 2):                                │
│                                                                 │
│    Terminal 1:  uvicorn app:app --reload     (FastAPI server)   │
│    Terminal 2:  celery -A celery_app worker  (Task executor)    │
│    Terminal 3:  redis-server                 (Message broker)   │
│                                                                 │
│                                                                 │
│  AFTER THIS LECTURE:                                            │
│                                                                 │
│    Terminal 1:  uvicorn app:app --reload     (FastAPI server)   │
│    Terminal 2:  celery -A celery_app worker  (Task executor)    │
│    Terminal 3:  redis-server                 (Message broker)   │
│    Terminal 4:  celery -A celery_app beat    (Scheduler) ← NEW  │
│    Terminal 5:  celery -A celery_app flower  (Monitor)   ← NEW  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Starting Beat:**

```bash
# Basic — start the Beat scheduler
celery -A celery_app beat --loglevel=info

# What you'll see in the terminal:
# celery beat v5.3.6 (emerald-rush) is starting.
# __    -    ... __   -        _
# LocalTime -> 2025-01-15 00:00:00
# Configuration ->
#     . broker -> redis://localhost:6379/0
#     . loader -> celery.loaders.app.AppLoader
#     . scheduler -> celery.beat.PersistentScheduler
#     . db -> celerybeat-schedule          ← schedule state file
#     . logfile -> [stderr]@%INFO
#     . maxinterval -> 5.00 minutes (300s)
```

**The schedule state file:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 BEAT STATE FILE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Beat creates a file called celerybeat-schedule in your         │
│  working directory. This file stores WHEN each task was         │
│  last sent.                                                     │
│                                                                 │
│  WHY IT MATTERS:                                                │
│  If you restart Beat, it reads this file and knows:             │
│  "I last sent cleanup-expired-tokens at 14:00.                  │
│  It's now 14:23. The schedule is every hour.                    │
│  Next send at 15:00 — not now."                                 │
│                                                                 │
│  Without this file, Beat would fire ALL tasks immediately       │
│  on restart — exactly the while-True problem from Part 1.       │
│                                                                 │
│  ┌──────────────────────────────────────────────┐               │
│  │ Add celerybeat-schedule to your .gitignore.  │               │
│  │ It is local state, not source code.          │               │
│  └──────────────────────────────────────────────┘               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Convenience: running Beat and worker in the same process (development only):**

```bash
# For local development, you can embed Beat inside the worker
celery -A celery_app worker --beat --loglevel=info

# The --beat flag makes the worker process ALSO run the scheduler.
# Convenient for development. NEVER use this in production.
# (We'll explain why in the next section.)
```

---

## 2.6 The Single-Beat Rule

**This is the most important operational rule in this entire section.**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ⚠️  THERE MUST BE EXACTLY ONE BEAT PROCESS. EVER.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why? Because Beat doesn't execute tasks — it sends messages.**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE DUPLICATE BEAT DISASTER                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ONE Beat (correct):                                            │
│                                                                 │
│    Beat ──── 00:00 ──── sends 1 message ──── 1 report runs     │
│                                                                 │
│                                                                 │
│  TWO Beats (disaster):                                          │
│                                                                 │
│    Beat 1 ── 00:00 ──── sends 1 message ──┐                    │
│                                            ├── 2 reports run!   │
│    Beat 2 ── 00:00 ──── sends 1 message ──┘                    │
│                                                                 │
│  Each Beat independently reads the schedule and sends           │
│  messages. They do not coordinate with each other.              │
│  Two Beats = every task runs twice. Three = three times.        │
│                                                                 │
│  Imagine your "generate invoice" task running twice.            │
│  Two invoices. Two charges to the customer.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This is why `--beat` on the worker is dangerous in production. If you scale to 3 workers with `--beat`, you get 3 Beat schedulers. Every scheduled task runs 3 times. In development, you run one worker, so it's fine. In production, run Beat as its own dedicated process."

**Factory analogy:**

> "A factory has one production schedule board and one shift manager reading it. If you accidentally hire three shift managers and give each a copy of the schedule, they'll all tell the machines to start the same job at the same time. You get three times the output you wanted — or three times the waste."

**How to enforce the single-Beat rule in production:**

```
┌─────────────────────────────────────────────────────────────────┐
│             ENFORCING SINGLE BEAT                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Option 1: Separate process (simplest)                          │
│  Run Beat as its own container/process in your deployment.      │
│  Scale workers to 5, 10, 50 — but only 1 Beat.                 │
│                                                                 │
│  Option 2: Pidfile                                              │
│  celery -A celery_app beat --pidfile=/var/run/celerybeat.pid    │
│  If a second Beat tries to start, it sees the pidfile and       │
│  refuses. Basic protection against accidental duplicates.       │
│                                                                 │
│  Option 3: Database-backed scheduler                            │
│  celery -A celery_app beat \                                    │
│    --scheduler django_celery_beat.schedulers:DatabaseScheduler   │
│  (or similar for non-Django projects)                           │
│  Stores schedule state in the database with a lock.             │
│  Safer in distributed deployments.                              │
│                                                                 │
│  For this course: Option 1. Separate process.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Timezone reminder:**

```python
# ALWAYS set your timezone explicitly
celery_app.conf.timezone = "UTC"

# If you don't set this, Beat uses the system timezone.
# Your laptop: America/New_York.
# The production server: UTC.
# Your "midnight" report now runs at different times
# depending on where the code is running. Set. The. Timezone.
```

---

# PART 3: TASK TIMEOUTS — SAFETY SWITCHES

## 3.1 Why Tasks Need Timeouts

**A task without a timeout is a machine without a safety switch.**

```python
# This task calls an external API.
# What if the API never responds?

@celery_app.task(bind=True, max_retries=3)
def sync_user_data(self, user_id: int) -> dict:
    # httpx has a timeout — but what if the task itself
    # has a logic error? An infinite loop? A deadlock?
    response = httpx.get(
        f"https://api.example.com/users/{user_id}",
        timeout=30,
    )
    
    # Or what if this database query hangs because of a lock?
    db.execute(update_user_query, response.json())
    
    return {"user_id": user_id, "synced": True}
```

> "You set a timeout on your HTTP request — good. But the HTTP timeout protects the *request*. It doesn't protect the *task*. What if the code before or after the request hangs? What if the database query waits on a lock forever? The task itself needs a ceiling on how long it can run."

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT HAPPENS WITHOUT TIMEOUTS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Worker has 4 worker processes (default concurrency).           │
│                                                                 │
│  13:00  Task A starts, hangs on DB lock    [█████████████████── │
│  13:01  Task B starts, hangs on API        [████████████████──  │
│  13:02  Task C starts, infinite loop bug   [███████████████──   │
│  13:03  Task D starts, hangs on DB lock    [██████████████──    │
│  13:04  Task E arrives...                                       │
│                                                                 │
│         ┌──────────────────────────────────────────────┐        │
│         │  All 4 processes are stuck.                  │        │
│         │  Task E sits in the queue. Waiting.          │        │
│         │  Task F, G, H... all pile up.                │        │
│         │  Your entire background system is FROZEN.    │        │
│         │  No scheduled tasks run. No user-triggered   │        │
│         │  tasks run. Everything stops.                │        │
│         │                                              │        │
│         │  And nothing in your logs tells you why.     │        │
│         └──────────────────────────────────────────────┘        │
│                                                                 │
│  Timeouts prevent one stuck task from killing the system.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Factory analogy:**

> "If a machine on the factory floor jams, it doesn't just stop making its product — it blocks the production line. Other products can't get to the next machine. A safety switch detects the jam and shuts down the machine so a technician can fix it and the line keeps moving."

---

## 3.2 soft_time_limit (The Warning Bell)

**`soft_time_limit` raises a special exception INSIDE the running task.**

The task is still alive. It can catch the exception, clean up resources, save partial progress, and exit gracefully.

```python
from celery.exceptions import SoftTimeLimitExceeded

@celery_app.task(soft_time_limit=120)  # 120 seconds = 2 minutes
def generate_large_report(report_type: str) -> dict:
    """Generate a report that might take a while."""
    
    partial_results = []
    
    try:
        for chunk in get_data_chunks(report_type):
            processed = process_chunk(chunk)
            partial_results.append(processed)
            
    except SoftTimeLimitExceeded:
        # ── THE WARNING BELL RANG ──────────────────────
        # We have a few seconds to clean up before the
        # hard time_limit kills us (if one is set).
        logger.warning(
            "report_soft_timeout",
            report_type=report_type,
            chunks_completed=len(partial_results),
        )
        # Save what we have so the work isn't lost
        save_partial_report(report_type, partial_results)
        return {
            "status": "partial",
            "chunks_completed": len(partial_results),
        }
    
    return {
        "status": "complete",
        "chunks_completed": len(partial_results),
    }
```

**How it works:**

```
┌─────────────────────────────────────────────────────────────────┐
│               soft_time_limit = 120                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Time:  0s         60s        120s       150s                   │
│         │           │          │          │                     │
│  Task:  [═══ working ════════]  │          │                     │
│                                │          │                     │
│                         SoftTimeLimitExceeded                   │
│                         raised INSIDE the task                  │
│                                │          │                     │
│                                [cleanup]  │                     │
│                                save data  │                     │
│                                log warning│                     │
│                                return     │                     │
│                                           │                     │
│  The task KEEPS RUNNING after the exception.                    │
│  It can catch the exception and do useful work.                 │
│  It should exit soon — the hard limit is coming.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What SoftTimeLimitExceeded actually is:**

```python
# SoftTimeLimitExceeded inherits from Exception.
# It is raised using a signal (SIGUSR1) from the worker
# parent process into the task's thread/process.

# You catch it exactly like any other exception.
# (Remember exception hierarchies from Week 1, Lecture 2.)

try:
    do_work()
except SoftTimeLimitExceeded:
    # Clean up gracefully
    ...
```

> "The soft limit is the warning bell on the factory floor. The machine buzzes: 'You've been running too long. Wrap it up.' The operator (your code) can hear the buzzer, save the current product, and shut down cleanly."

---

## 3.3 time_limit (The Emergency Stop)

**`time_limit` kills the task's worker process. No exceptions. No cleanup. Dead.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  time_limit = 180                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Time:  0s         120s             180s                        │
│         │           │                │                          │
│  Task:  [═══ working ═══════════════]                           │
│                                      │                          │
│                                    SIGKILL                      │
│                                  Worker process                 │
│                                  terminated.                    │
│                                      │                          │
│                                      ▼                          │
│                                  New worker                     │
│                                  process spawned                │
│                                  automatically.                 │
│                                                                 │
│  The task CANNOT catch this. It is an OS-level kill.            │
│  The worker's parent process detects the dead child             │
│  and spawns a replacement. The task is marked FAILURE.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The critical difference:**

```
┌─────────────────────────────────────────────────────────────────┐
│            soft_time_limit VS time_limit                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    soft_time_limit        time_limit             │
│                    ───────────────        ──────────             │
│  Mechanism:        Raises exception       Sends SIGKILL         │
│  Task can catch:   Yes ✅                 No ❌                  │
│  Cleanup possible: Yes ✅                 No ❌                  │
│  Task survives:    Yes (if caught)        No (process dies)     │
│  Analogy:          Warning buzzer         Emergency power cut   │
│                                                                 │
│                                                                 │
│  RECOMMENDED PATTERN: Use both together.                        │
│                                                                 │
│      soft_time_limit = X    ← Give task a chance to clean up   │
│      time_limit = X + 30    ← Hard kill if cleanup hangs too   │
│                                                                 │
│                                                                 │
│  Time:  0        soft(X)    hard(X+30)                          │
│         │           │           │                               │
│  Task:  [══ work ══]           │                                │
│                     [cleanup]  │                                │
│                         │      │                                │
│                         ▼      │                                │
│                     exits OK   │                                │
│                                │                                │
│         OR if cleanup hangs:   │                                │
│                     [cleanup ──────]                             │
│                                │                                │
│                             SIGKILL  ← last resort              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Setting Timeouts (Per-Task and Global)

**Per-task (preferred — each task has different expected durations):**

```python
# Quick task — should finish in under 30 seconds
@celery_app.task(soft_time_limit=30, time_limit=60)
def check_api_health(timeout: int = 10) -> dict:
    ...

# Medium task — may take a few minutes
@celery_app.task(soft_time_limit=300, time_limit=330)
def generate_daily_report(report_type: str) -> dict:
    ...

# Long task — batch processing, may take up to 30 minutes
@celery_app.task(soft_time_limit=1800, time_limit=1860)
def archive_completed_tasks(days_old: int) -> int:
    ...
```

**Global defaults (safety net for all tasks):**

```python
# celery_app.py — add to your configuration

celery_app.conf.task_soft_time_limit = 300   # 5 minutes default
celery_app.conf.task_time_limit = 330        # 5.5 minutes hard kill

# Per-task settings OVERRIDE these globals.
# Globals exist to catch tasks where you FORGOT to set a timeout.
# Every task should have an explicit timeout, but humans forget.
# The global default is your safety net.
```

**How to think about choosing timeout values:**

```
┌─────────────────────────────────────────────────────────────────┐
│             CHOOSING TIMEOUT VALUES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Measure how long the task NORMALLY takes.                   │
│     Run it 10 times. Note the slowest run.                      │
│                                                                 │
│  2. Set soft_time_limit to 2-3x the normal worst case.          │
│     If it normally takes 10s at worst → soft limit = 30s.       │
│     This gives room for slow networks, heavy DB load, etc.      │
│                                                                 │
│  3. Set time_limit to soft_time_limit + cleanup time.           │
│     If cleanup takes at most 10s → time_limit = 40s.            │
│                                                                 │
│  4. If you can't measure yet, start generous and tighten.       │
│     Better to start with soft=300, hard=330 and collect         │
│     data, than to start with soft=5 and kill healthy tasks.     │
│                                                                 │
│  ┌────────────────────────────────────────────────────┐         │
│  │  A timeout that's too tight is WORSE than no       │         │
│  │  timeout at all. Tight timeouts kill healthy tasks  │         │
│  │  under load, masking the real problem.             │         │
│  └────────────────────────────────────────────────────┘         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Graceful Cleanup on Timeout

**A complete production pattern combining both limits:**

```python
from celery.exceptions import SoftTimeLimitExceeded
import structlog

logger = structlog.get_logger()

@celery_app.task(
    bind=True,
    soft_time_limit=120,
    time_limit=150,
)
def sync_user_data(self, user_id: int) -> dict:
    """
    Sync user data from external API to our database.
    
    Timeout strategy:
    - soft_time_limit=120: After 2 min, get a chance to clean up.
    - time_limit=150: After 2.5 min, hard kill. This gives 30s
      for the cleanup code in the except block.
    """
    log = logger.bind(task_id=self.request.id, user_id=user_id)
    db_session = None
    
    try:
        log.info("sync_started")
        
        # Step 1: Fetch from external API
        external_data = fetch_from_external_api(user_id)
        
        # Step 2: Transform
        transformed = transform_user_data(external_data)
        
        # Step 3: Write to database
        db_session = get_db_session()
        update_user_in_db(db_session, user_id, transformed)
        db_session.commit()
        
        log.info("sync_completed")
        return {"user_id": user_id, "status": "synced"}
    
    except SoftTimeLimitExceeded:
        log.warning("sync_soft_timeout")
        
        # Roll back any uncommitted DB work
        if db_session is not None:
            db_session.rollback()
        
        # Return a meaningful result so the caller knows
        return {"user_id": user_id, "status": "timeout"}
    
    except Exception as exc:
        log.error("sync_failed", error=str(exc))
        
        if db_session is not None:
            db_session.rollback()
        
        raise  # Let Celery's retry mechanism handle it
    
    finally:
        # Always close the session
        # (Remember context managers from Week 1 — same idea,
        #  but we need manual control here because the task
        #  lifecycle is different from a with block.)
        if db_session is not None:
            db_session.close()
```

**The execution timeline for this task:**

```
┌─────────────────────────────────────────────────────────────────┐
│         HAPPY PATH               TIMEOUT PATH                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Happy (finishes in 45s):    Timeout (API hangs):               │
│                                                                 │
│  0s   sync_started           0s   sync_started                  │
│  │    fetch API (20s)        │    fetch API ...                  │
│  │    transform (5s)         │    ... waiting ...                │
│  │    DB write (10s)         │    ... waiting ...                │
│  │    commit (1s)            │    ... waiting ...                │
│  45s  sync_completed         120s SoftTimeLimitExceeded          │
│       return "synced"        │    rollback DB                   │
│       ✅ done                │    return "timeout"              │
│                              125s ✅ done (cleanup took 5s)     │
│                                                                 │
│                              OR if cleanup also hangs:          │
│                              120s SoftTimeLimitExceeded          │
│                              │    rollback hangs...             │
│                              │    ... stuck ...                 │
│                              150s SIGKILL 💀                    │
│                                   process killed                │
│                                   new worker spawns             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: DEAD LETTER HANDLING

## 4.1 What is a Poison Message?

**A poison message is a task that ALWAYS fails, no matter how many times you retry it.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   POISON MESSAGES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TRANSIENT FAILURE (retries help):                              │
│  ──────────────────────────────────                             │
│  Attempt 1: API timeout   → retry                              │
│  Attempt 2: API timeout   → retry                              │
│  Attempt 3: API responds! → success ✅                          │
│                                                                 │
│  The failure was temporary. Network blip. Server busy.          │
│  Retries (from Lecture 2) solve this perfectly.                 │
│                                                                 │
│                                                                 │
│  POISON MESSAGE (retries are useless):                          │
│  ──────────────────────────────────────                         │
│  Attempt 1: User ID 99999 doesn't exist in external API → fail │
│  Attempt 2: User ID 99999 doesn't exist in external API → fail │
│  Attempt 3: User ID 99999 doesn't exist in external API → fail │
│  All retries exhausted → FAILURE ❌                             │
│                                                                 │
│  The failure is PERMANENT. Retrying the same input with the     │
│  same code will always produce the same error. The message      │
│  is poisoned — it can never succeed.                            │
│                                                                 │
│  Other examples:                                                │
│  • Invalid data format (bad JSON in the arguments)              │
│  • Permission revoked (API key expired permanently)             │
│  • Database constraint violation (duplicate key)                │
│  • Bug in the task code itself                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Factory analogy:**

> "A defective raw material arrives at the machine. The machine tries to process it — fails. Quality control sends it back. The machine tries again — fails. QC sends it back again. After three attempts, QC doesn't send it back a fourth time. They put it in the **reject bin** for a human to inspect. Maybe the raw material was wrong. Maybe the machine needs recalibration. Either way, the reject bin prevents the machine from spinning its wheels on something it can never fix."

---

## 4.2 When Retries Aren't Enough

**Recall from Lecture 2 — your retry configuration:**

```python
# This is what you already know.
@celery_app.task(
    bind=True,
    max_retries=3,
    autoretry_for=(ConnectionError, TimeoutError),
    retry_backoff=True,
)
def unreliable_api_call(self, url: str) -> dict:
    response = httpx.get(url, timeout=10)
    response.raise_for_status()
    return response.json()
```

**What happens after max_retries is exhausted?**

```
┌─────────────────────────────────────────────────────────────────┐
│           AFTER MAX RETRIES: WHAT CELERY DOES                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Attempt 1: ConnectionError → retry (wait 1s)                  │
│  Attempt 2: ConnectionError → retry (wait 2s)                  │
│  Attempt 3: ConnectionError → retry (wait 4s)                  │
│  Attempt 4: ConnectionError → MAX RETRIES REACHED              │
│                                                                 │
│  Celery does the following:                                     │
│  1. Marks the task state as FAILURE                             │
│  2. Stores the exception in the result backend (Redis)          │
│  3. Calls the task's on_failure() method (if defined)           │
│  4. Moves on                                                    │
│                                                                 │
│  What Celery does NOT do:                                       │
│  ❌ Send you an email                                           │
│  ❌ Log a special warning                                       │
│  ❌ Store the failed task for manual review                     │
│  ❌ Alert anyone                                                │
│                                                                 │
│  The task quietly dies. If you aren't checking the result       │
│  backend, you will never know it failed.                        │
│                                                                 │
│  This is the "silent failure at 3 AM" from Part 1.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Celery's default behavior after exhausting retries is: mark it as failed and forget about it. That's fine for tasks where failure is acceptable — a cache refresh that'll run again in 5 minutes anyway. But for tasks where failure means **lost data** or **broken business logic**, you need to explicitly handle the dead letter."

---

## 4.3 The on_failure Hook

**Every Celery task has lifecycle hooks. `on_failure` fires when the task fails after all retries.**

```python
@celery_app.task(
    bind=True,
    max_retries=3,
    autoretry_for=(ConnectionError, TimeoutError),
    retry_backoff=True,
)
def sync_user_data(self, user_id: int) -> dict:
    response = httpx.get(
        f"https://api.example.com/users/{user_id}",
        timeout=10,
    )
    response.raise_for_status()
    return response.json()
```

**Option 1 — Override on_failure directly on the task:**

```python
@celery_app.task(
    bind=True,
    max_retries=3,
    autoretry_for=(ConnectionError, TimeoutError),
    retry_backoff=True,
)
def sync_user_data(self, user_id: int) -> dict:
    response = httpx.get(
        f"https://api.example.com/users/{user_id}",
        timeout=10,
    )
    response.raise_for_status()
    return response.json()

# This method is defined OUTSIDE the task function,
# but on the same task object — using the base class.
# It's cleaner to use a custom base class (shown next).
```

**Option 2 (preferred) — Custom base task class:**

```python
# base_task.py
import structlog
from celery import Task

logger = structlog.get_logger()


class AlertingTask(Task):
    """
    Custom base task that handles failures after all retries
    are exhausted. Use this as the base for any task where
    silent failure is not acceptable.
    """
    
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        """
        Called when the task fails permanently.
        
        Parameters (provided by Celery — you don't pass these):
            exc:     The exception that caused the failure
            task_id: Unique ID of this task execution
            args:    Positional arguments the task was called with
            kwargs:  Keyword arguments the task was called with
            einfo:   ExceptionInfo object (contains traceback)
        """
        logger.error(
            "task_dead_letter",
            task_name=self.name,
            task_id=task_id,
            args=args,
            kwargs=kwargs,
            error=str(exc),
            error_type=type(exc).__name__,
            traceback=str(einfo),
        )
        
        # Call the parent to preserve default behavior
        super().on_failure(exc, task_id, args, kwargs, einfo)
```

**Using the custom base:**

```python
# tasks.py
from base_task import AlertingTask

@celery_app.task(
    base=AlertingTask,    # ← Use our custom base
    bind=True,
    max_retries=3,
    autoretry_for=(ConnectionError, TimeoutError),
    retry_backoff=True,
)
def sync_user_data(self, user_id: int) -> dict:
    response = httpx.get(
        f"https://api.example.com/users/{user_id}",
        timeout=10,
    )
    response.raise_for_status()
    return response.json()


# Now when sync_user_data fails after 3 retries,
# AlertingTask.on_failure fires automatically.
# Every task using base=AlertingTask gets this behavior.
```

**What `on_failure` gives you:**

```
┌─────────────────────────────────────────────────────────────────┐
│              on_failure PARAMETERS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  exc      The exception instance.                               │
│           You can check its type to decide what to do.          │
│           isinstance(exc, NotFoundError) → different from       │
│           isinstance(exc, ConnectionError)                      │
│                                                                 │
│  task_id  The unique ID of this specific execution.             │
│           "abc123-def456-..."                                    │
│           Useful for tracing: "which run failed?"               │
│                                                                 │
│  args     The positional args the task was called with.         │
│           sync_user_data.delay(42) → args = (42,)              │
│           You know WHAT input caused the failure.               │
│                                                                 │
│  kwargs   The keyword args. Same idea.                          │
│                                                                 │
│  einfo    Contains the full traceback.                          │
│           str(einfo) gives you the formatted traceback          │
│           string — same as what you'd see in the terminal.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Building a Dead Letter Store

**Logging is not enough. Logs scroll off the screen and get rotated. For tasks where failure matters, store the dead letter in a place humans can review it.**

```python
# dead_letters.py
"""
Dead letter storage.

A dead letter record contains everything needed to understand
what failed and to retry it manually later.
"""

from datetime import datetime
from sqlalchemy import Column, Integer, String, Text, DateTime
from sqlalchemy.dialects.postgresql import JSONB
from database import Base  # Your existing SQLAlchemy Base


class DeadLetter(Base):
    """
    Stores tasks that failed permanently after all retries.
    
    This table is an inspection queue for humans and admin
    dashboards. When on-call engineers check the system each
    morning, they review this table.
    """
    __tablename__ = "dead_letters"

    id = Column(Integer, primary_key=True)
    task_name = Column(String, nullable=False, index=True)
    task_id = Column(String, nullable=False, unique=True)
    args = Column(JSONB, default=list)
    kwargs = Column(JSONB, default=dict)
    exception_type = Column(String, nullable=False)
    exception_message = Column(Text, nullable=False)
    traceback = Column(Text)
    failed_at = Column(DateTime, default=datetime.utcnow, index=True)
    resolved = Column(
        String, 
        default="pending",  # "pending", "retried", "ignored"
        index=True,
    )
```

**Now update the base task to store dead letters:**

```python
# base_task.py — extended
import structlog
from celery import Task
from datetime import datetime
from dead_letters import DeadLetter
from database import SyncSessionLocal  # Sync session — see note below

logger = structlog.get_logger()


class AlertingTask(Task):
    """
    Base task that stores dead letters in PostgreSQL
    when all retries are exhausted.
    """
    
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        # ── 1. LOG IT ──────────────────────────────────
        logger.error(
            "task_dead_letter",
            task_name=self.name,
            task_id=task_id,
            args=args,
            error_type=type(exc).__name__,
            error=str(exc),
        )
        
        # ── 2. STORE IT ───────────────────────────────
        # NOTE: Celery workers can use sync DB sessions.
        # The worker is already a separate process —
        # it's not your async FastAPI event loop.
        try:
            session = SyncSessionLocal()
            dead_letter = DeadLetter(
                task_name=self.name,
                task_id=task_id,
                args=list(args) if args else [],
                kwargs=dict(kwargs) if kwargs else {},
                exception_type=type(exc).__name__,
                exception_message=str(exc),
                traceback=str(einfo),
                failed_at=datetime.utcnow(),
            )
            session.add(dead_letter)
            session.commit()
        except Exception as store_exc:
            # If even storing the dead letter fails, 
            # at least the log entry above exists.
            logger.error(
                "dead_letter_store_failed",
                original_task=self.name,
                store_error=str(store_exc),
            )
        finally:
            session.close()
        
        # ── 3. CALL PARENT ─────────────────────────────
        super().on_failure(exc, task_id, args, kwargs, einfo)
```

**Why we use a sync session in the worker (the note above):**

```
┌─────────────────────────────────────────────────────────────────┐
│            SYNC vs ASYNC IN CELERY WORKERS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your FastAPI app uses ASYNC SQLAlchemy (AsyncSession)          │
│  because FastAPI runs in an async event loop.                   │
│                                                                 │
│  Celery workers are SEPARATE PROCESSES with their own           │
│  runtime. The on_failure hook runs in a synchronous context     │
│  inside the worker. Using sync SQLAlchemy here is correct       │
│  and simpler.                                                   │
│                                                                 │
│  FastAPI process → AsyncSession (async event loop)              │
│  Celery worker   → SyncSession (separate process, no loop)     │
│                                                                 │
│  You can use async in Celery workers (it's possible), but       │
│  for hooks like on_failure that do a simple DB insert,          │
│  sync is easier and has no drawbacks.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The full lifecycle with dead letter handling:**

```
┌─────────────────────────────────────────────────────────────────┐
│            TASK LIFECYCLE WITH DEAD LETTERS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Task called                                                    │
│     │                                                           │
│     ▼                                                           │
│  Attempt 1 ── fails ── autoretry (wait 1s)                      │
│     │                                                           │
│     ▼                                                           │
│  Attempt 2 ── fails ── autoretry (wait 2s)                      │
│     │                                                           │
│     ▼                                                           │
│  Attempt 3 ── fails ── autoretry (wait 4s)                      │
│     │                                                           │
│     ▼                                                           │
│  Attempt 4 ── fails ── max_retries reached!                     │
│     │                                                           │
│     ├───▶ on_failure() fires                                    │
│     │     ├─ Log error with structlog                           │
│     │     ├─ Store DeadLetter in PostgreSQL                     │
│     │     └─ (Later: send alert — Part 6)                       │
│     │                                                           │
│     ▼                                                           │
│  Task state = FAILURE in result backend                         │
│  Worker moves on to next task                                   │
│                                                                 │
│  LATER: Human reviews dead_letters table                        │
│     ├─ Finds the record                                         │
│     ├─ Investigates: "Why did user_id=99999 fail?"              │
│     ├─ Fix: "That user was deleted from external system"        │
│     └─ Mark as "resolved" or manually re-queue                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: MONITORING WITH FLOWER

## 5.1 Why Monitoring Matters

> "You now have Beat scheduling tasks, timeouts protecting workers, and dead letter handling catching permanent failures. But imagine managing all of this by reading raw log output in five terminal windows. At 3 AM. After a page wakes you up."

```
┌─────────────────────────────────────────────────────────────────┐
│             MONITORING = THE CONTROL ROOM                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Without monitoring, you're a factory owner who locked          │
│  the control room door and threw away the key.                  │
│                                                                 │
│  You can hear the machines running.                             │
│  You can't see which ones.                                      │
│  You can't see the queue of raw materials piling up.            │
│  You can't see the reject bin overflowing.                      │
│  You find out something broke when customers complain.          │
│                                                                 │
│  QUESTIONS MONITORING ANSWERS:                                  │
│  ├─ How many tasks ran in the last hour? Succeeded? Failed?     │
│  ├─ Are any workers down? Overloaded?                           │
│  ├─ How big is the queue? Are tasks piling up?                  │
│  ├─ Which task is taking the longest?                           │
│  ├─ What was the error for task abc-123?                        │
│  └─ Is Beat sending tasks on schedule?                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Flower Setup

**Flower is a real-time web UI for Celery. It connects to the same broker your workers use and displays everything that is happening.**

```bash
# Install (add to your requirements.txt / pyproject.toml)
pip install flower

# Run it (separate terminal)
celery -A celery_app flower --port=5555

# That's it. Open http://localhost:5555 in your browser.
```

**How Flower connects:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  FLOWER ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌───────────────┐                            │
│                    │   FLOWER      │                            │
│                    │  :5555        │                            │
│                    │ (web UI)      │                            │
│                    └───────┬───────┘                            │
│                            │ reads events                      │
│                            ▼                                    │
│    ┌──────────┐     ┌───────────┐     ┌──────────┐             │
│    │   BEAT   │────▶│   REDIS   │◀────│ WORKER 1 │             │
│    │(scheduler│     │  (broker) │     │          │             │
│    └──────────┘     │           │     └──────────┘             │
│                     │           │     ┌──────────┐             │
│                     │           │◀────│ WORKER 2 │             │
│                     └───────────┘     │          │             │
│                                       └──────────┘             │
│                                                                 │
│  Flower does NOT intercept or modify task execution.            │
│  It is read-only. It subscribes to Celery's event stream       │
│  and displays the data in a web interface.                      │
│  Adding or removing Flower has zero effect on your workers.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**To get full event data, enable events on your workers:**

```bash
# Start worker with events enabled
celery -A celery_app worker --loglevel=info -E

# The -E flag (or --events) tells the worker to broadcast
# lifecycle events: task-sent, task-received, task-started,
# task-succeeded, task-failed, task-retried, etc.
# Flower listens for these events.
```

---

## 5.3 Reading the Dashboard

**Flower's web interface has several pages. Here's what each one shows and why you care:**

```
┌─────────────────────────────────────────────────────────────────┐
│              FLOWER DASHBOARD LAYOUT                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  🏠 Dashboard        Workers   Tasks   Broker   Monitor   │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │                                                           │  │
│  │  Active Tasks: 3          Processed: 1,247                │  │
│  │  Reserved: 12             Failed: 23                      │  │
│  │  Workers: 2 online        Success Rate: 98.2%             │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│                                                                 │
│  PAGE: Workers                                                  │
│  ─────────────                                                  │
│  Shows each worker process:                                     │
│  • Name, status (online/offline)                                │
│  • Concurrency (how many task slots)                            │
│  • Active tasks (what it's doing right now)                     │
│  • Tasks completed (total count)                                │
│  • Load average                                                 │
│                                                                 │
│  WHY YOU CARE: If a worker disappears from this list,           │
│  it crashed. If active tasks = concurrency, it's at capacity.   │
│                                                                 │
│                                                                 │
│  PAGE: Tasks                                                    │
│  ───────────                                                    │
│  Shows individual task executions:                              │
│  • Task name, ID, state (PENDING, STARTED, SUCCESS, FAILURE)   │
│  • Arguments it was called with                                 │
│  • Worker that executed it                                      │
│  • Start time, runtime duration                                 │
│  • Exception + traceback (for failed tasks)                     │
│  • Result (for successful tasks)                                │
│                                                                 │
│  WHY YOU CARE: When something breaks, you open this page,       │
│  filter by FAILURE, and read the traceback. Faster than         │
│  digging through log files.                                     │
│                                                                 │
│                                                                 │
│  PAGE: Broker                                                   │
│  ────────────                                                   │
│  Shows the Redis queue status:                                  │
│  • Queue names and message counts                               │
│  • Messages in each queue (backlog size)                        │
│                                                                 │
│  WHY YOU CARE: If the "celery" queue shows 5,000 pending        │
│  messages and it's growing, tasks are arriving faster than       │
│  workers can process them. You need more workers or faster       │
│  tasks.                                                         │
│                                                                 │
│                                                                 │
│  PAGE: Monitor                                                  │
│  ─────────────                                                  │
│  Real-time graphs:                                              │
│  • Tasks succeeded/failed over time                             │
│  • Task execution times                                         │
│  • Queue length over time                                       │
│                                                                 │
│  WHY YOU CARE: Trends. A slowly growing failure rate means       │
│  a degrading external dependency. A sudden spike in queue       │
│  length means something triggered a flood of tasks.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What to look for — the key indicators:**

```
┌─────────────────────────────────────────────────────────────────┐
│              FLOWER: WHAT HEALTHY LOOKS LIKE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ HEALTHY:                                                    │
│  • All workers show "online"                                    │
│  • Active tasks < concurrency (workers have spare capacity)     │
│  • Queue size near 0 or stable (not growing)                    │
│  • Failure rate < 2-5%                                          │
│  • Task durations consistent (no sudden spikes)                 │
│                                                                 │
│                                                                 │
│  ⚠️  WARNING SIGNS:                                             │
│  • Queue size growing steadily → tasks arriving too fast        │
│  • Active tasks = concurrency on all workers → at capacity      │
│  • Failure rate rising → external dependency degrading          │
│  • One task type dominating active list → might be hanging      │
│                                                                 │
│                                                                 │
│  🔴 SOMETHING IS WRONG:                                         │
│  • Worker disappeared → process crashed                         │
│  • Queue at 10,000+ and growing → backlog crisis                │
│  • 100% failure rate on one task → poison message or            │
│    complete outage of dependency                                │
│  • Same task_id active for hours → stuck (no timeout set!)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 Worker Inspection from Code

**You don't always need the Flower UI. Celery provides an inspect API for programmatic checks. This is useful for health endpoints in your FastAPI application.**

```python
# worker_inspection.py
"""
Programmatic inspection of Celery workers.
Use this in health check endpoints and alerting scripts.
"""

from celery_app import celery_app


def get_worker_status() -> dict:
    """
    Check which workers are alive and what they're doing.
    
    This sends a broadcast message to all workers via the
    broker and collects their responses. It has a timeout —
    if a worker doesn't respond, it's probably dead.
    """
    inspector = celery_app.control.inspect(timeout=5.0)
    
    # ── Who's online? ──────────────────────────────────
    ping_responses = inspector.ping()
    # Returns: {'worker1@host': {'ok': 'pong'}, ...}
    # or None if no workers respond
    
    if ping_responses is None:
        return {
            "status": "critical",
            "workers_online": 0,
            "detail": "No workers responded to ping",
        }
    
    # ── What's running right now? ──────────────────────
    active_tasks = inspector.active()
    # Returns: {'worker1@host': [{'id': '...', 'name': '...', ...}]}
    
    # ── What's waiting in each worker's local buffer? ──
    reserved_tasks = inspector.reserved()
    # Returns tasks that the worker has fetched from the
    # broker but hasn't started executing yet.
    
    # ── Build summary ──────────────────────────────────
    workers = list(ping_responses.keys())
    total_active = sum(
        len(tasks) for tasks in (active_tasks or {}).values()
    )
    total_reserved = sum(
        len(tasks) for tasks in (reserved_tasks or {}).values()
    )
    
    return {
        "status": "healthy" if workers else "critical",
        "workers_online": len(workers),
        "workers": workers,
        "active_tasks": total_active,
        "reserved_tasks": total_reserved,
    }
```

**Hook this into your FastAPI application:**

```python
# routes/health.py
from fastapi import APIRouter, HTTPException
from worker_inspection import get_worker_status

router = APIRouter(tags=["health"])

@router.get("/health/workers")
async def check_workers():
    """
    Health check endpoint for Celery workers.
    
    Returns 200 if workers are online, 503 if not.
    Load balancers and uptime monitors hit this endpoint.
    """
    # NOTE: inspector.ping() is a blocking call.
    # In production, run this in a thread pool to avoid
    # blocking the async event loop.
    # For now, this is sufficient to learn the pattern.
    status = get_worker_status()
    
    if status["status"] != "healthy":
        raise HTTPException(
            status_code=503,
            detail=status,
        )
    
    return status
```

**What the endpoint returns:**

```
# Workers healthy:
GET /health/workers → 200
{
    "status": "healthy",
    "workers_online": 2,
    "workers": ["worker1@hostname", "worker2@hostname"],
    "active_tasks": 3,
    "reserved_tasks": 7
}

# No workers:
GET /health/workers → 503
{
    "status": "critical",
    "workers_online": 0,
    "detail": "No workers responded to ping"
}
```

---

# PART 6: ALERTING ON FAILURES

## 6.1 The Dashboard Nobody Watches

> "Flower is great. You have a beautiful dashboard. Graphs, tables, real-time updates. There's just one problem."

```
┌─────────────────────────────────────────────────────────────────┐
│           THE DASHBOARD NOBODY WATCHES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Friday 5 PM:   You check Flower. Everything green. ✅          │
│  Friday 8 PM:   External API changes their response format.    │
│  Friday 8:01 PM: Every parse_api_response task starts failing. │
│  Saturday:       Failures accumulate. Nobody's looking.         │
│  Sunday:         Queue backed up. Beat keeps sending tasks.     │
│  Monday 9 AM:    You open Flower. 4,000 failed tasks.          │
│                  Two days of data not synced.                   │
│                                                                 │
│  MONITORING without ALERTING is just a prettier log file.       │
│  The system must TELL YOU when something is wrong.              │
│  You should not have to remember to look.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Factory analogy:**

> "A factory control room with screens and no alarms is a museum exhibit. The screens exist so that when the alarm goes off, the operator can look at them and figure out what happened. The alarm is the important part."

---

## 6.2 Celery Signals

**Celery emits signals at every stage of a task's lifecycle. You can attach handler functions to these signals — like attaching event listeners to DOM elements in JavaScript, but for task events.**

```python
# signals.py
"""
Celery signal handlers for alerting and observability.

Import this module in your celery_app.py to activate the handlers.
Signals fire automatically — you don't call these functions.
"""

import structlog
from celery.signals import (
    task_failure,
    task_success,
    task_retry,
    worker_ready,
    worker_shutting_down,
)

logger = structlog.get_logger()
```

**The task_failure signal — the most important one:**

```python
@task_failure.connect
def handle_task_failure(
    sender,       # The task class (not instance)
    task_id,      # Unique execution ID
    exception,    # The exception object
    args,         # Positional args the task was called with
    kwargs,       # Keyword args
    traceback,    # Traceback object
    einfo,        # ExceptionInfo with formatted traceback
    **kw,         # Always accept **kw for forward compatibility
):
    """
    Fires EVERY time a task fails — including each retry
    AND the final failure.
    
    Use this for centralized alerting across ALL tasks,
    not just tasks with base=AlertingTask.
    """
    logger.error(
        "task_failed",
        task_name=sender.name,
        task_id=task_id,
        error_type=type(exception).__name__,
        error=str(exception),
    )
    
    # You could send a Slack webhook, email, PagerDuty alert, etc.
    # Be careful: this fires on EVERY failure, including retries.
    # You probably don't want an alert for every retry attempt.
    # Filter by checking if retries are exhausted:
    
    # Only alert on FINAL failure (after all retries):
    task = sender
    request = kw.get("request", None)
    # Unfortunately, the signal doesn't directly tell you if
    # retries are exhausted. The on_failure hook (Part 4) is
    # better for final-failure-only alerting.
    # Use the signal for LOGGING; use on_failure for ALERTING.
```

**The difference between signals and on_failure:**

```
┌─────────────────────────────────────────────────────────────────┐
│         SIGNAL (task_failure) vs HOOK (on_failure)             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  task_failure SIGNAL:                                           │
│  ─────────────────────                                          │
│  • Fires on EVERY failure (including each retry attempt)        │
│  • Defined ONCE, applies to ALL tasks (centralized)             │
│  • Good for: logging, metrics, counters                         │
│  • Bad for: alerting (too noisy — fires on retries)             │
│                                                                 │
│  on_failure HOOK (from Part 4):                                 │
│  ──────────────────────────────                                 │
│  • Fires ONLY on final failure (after all retries exhausted)    │
│  • Defined per task class (via base=AlertingTask)               │
│  • Good for: alerting, dead letter storage                      │
│  • You choose which tasks get this behavior                     │
│                                                                 │
│                                                                 │
│  RECOMMENDED PATTERN:                                           │
│  ─────────────────────                                          │
│  ┌──────────────────────────────────────────────────┐           │
│  │  Use task_failure signal for centralized LOGGING. │           │
│  │  Use on_failure hook for targeted ALERTING.       │           │
│  └──────────────────────────────────────────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Other useful signals:**

```python
@task_retry.connect
def handle_task_retry(sender, request, reason, einfo, **kw):
    """
    Fires when a task is about to be retried.
    Useful for tracking retry rates.
    """
    logger.warning(
        "task_retrying",
        task_name=sender.name,
        task_id=request.id,
        retry_reason=str(reason),
    )


@worker_ready.connect
def handle_worker_ready(sender, **kw):
    """
    Fires when a worker has finished starting up and is 
    ready to accept tasks. Useful for deployment verification.
    """
    logger.info("worker_online", worker=str(sender))


@worker_shutting_down.connect
def handle_worker_shutdown(sender, sig, how, exitcode, **kw):
    """
    Fires when a worker begins shutting down.
    'how' is 'warm' (finish current tasks) or 'cold' (immediate).
    """
    logger.info(
        "worker_shutting_down",
        worker=str(sender),
        signal=sig,
        how=how,
    )
```

**Activating signals — make sure they're imported:**

```python
# celery_app.py — add this at the bottom

# Import signals module so handlers are registered.
# If you don't import it, the @signal.connect decorators
# never execute, and no handlers are attached.
import signals  # noqa: F401  (imported for side effect)
```

---

## 6.3 Health Check Endpoints

**Combine the worker inspection (Part 5) with queue depth checking for a comprehensive health endpoint:**

```python
# routes/health.py — extended
from fastapi import APIRouter, HTTPException
from celery_app import celery_app
from worker_inspection import get_worker_status

router = APIRouter(tags=["health"])


@router.get("/health/background")
async def background_system_health():
    """
    Comprehensive health check for the background task system.
    Checks: workers alive, queue depth, recent failure rate.
    """
    report = {}
    
    # ── 1. Are workers running? ────────────────────────
    worker_status = get_worker_status()
    report["workers"] = worker_status
    
    # ── 2. How deep is the queue? ──────────────────────
    # Use the broker connection to check queue length
    with celery_app.connection_or_acquire() as conn:
        queue_length = conn.default_channel.queue_declare(
            queue="celery", passive=True
        ).message_count
    
    report["queue_depth"] = queue_length
    
    # ── 3. Determine overall status ────────────────────
    if worker_status["workers_online"] == 0:
        report["status"] = "critical"
        report["reason"] = "No workers online"
        raise HTTPException(status_code=503, detail=report)
    
    if queue_length > 1000:
        report["status"] = "degraded"
        report["reason"] = f"Queue backlog: {queue_length} tasks"
        # Return 200 still — workers are running, just busy
    else:
        report["status"] = "healthy"
    
    return report
```

**What the health check tells you:**

```
┌─────────────────────────────────────────────────────────────────┐
│              HEALTH CHECK RESPONSES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HEALTHY — everything normal                                    │
│  {"status": "healthy", "workers": {"workers_online": 2, ...},  │
│   "queue_depth": 3}                                             │
│                                                                 │
│  DEGRADED — working but stressed                                │
│  {"status": "degraded", "reason": "Queue backlog: 2341 tasks", │
│   "workers": {"workers_online": 2, ...}, "queue_depth": 2341}  │
│                                                                 │
│  CRITICAL — broken, return 503                                  │
│  {"status": "critical", "reason": "No workers online",         │
│   "workers": {"workers_online": 0, ...}, "queue_depth": 8422}  │
│                                                                 │
│  An uptime monitor (or load balancer) hits this endpoint        │
│  periodically. If it gets 503, it pages the on-call engineer.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.4 Common Mistakes

### Mistake 1: Running multiple Beat instances

```
# ❌ WRONG: Starting 3 workers with --beat
celery -A celery_app worker --beat --concurrency=4  # instance 1
celery -A celery_app worker --beat --concurrency=4  # instance 2
celery -A celery_app worker --beat --concurrency=4  # instance 3

# Result: every scheduled task runs 3 times.
# Triple the database writes, triple the API calls,
# triple the emails sent to customers.

# ✅ CORRECT: One dedicated Beat, scale workers separately
celery -A celery_app beat                            # ONE Beat
celery -A celery_app worker --concurrency=4          # Worker 1
celery -A celery_app worker --concurrency=4          # Worker 2
celery -A celery_app worker --concurrency=4          # Worker 3
```

---

### Mistake 2: No timeouts anywhere

```python
# ❌ WRONG: No timeout set on a task that calls an external API
@celery_app.task
def sync_all_users():
    for user in get_all_users():      # 10,000 users
        httpx.get(f"https://api.example.com/users/{user.id}")
        # If the API hangs, this task runs FOREVER.
        # It holds a worker slot FOREVER.
        # Beat sends a new sync_all_users every day.
        # Now TWO tasks are hanging. Then three. Then four.
        # All worker slots consumed. System frozen.

# ✅ CORRECT: Always set timeouts
@celery_app.task(soft_time_limit=1800, time_limit=1860)
def sync_all_users():
    for user in get_all_users():
        httpx.get(
            f"https://api.example.com/users/{user.id}",
            timeout=10,  # HTTP timeout too
        )
```

---

### Mistake 3: Alerting on every retry

```python
# ❌ WRONG: Send a Slack message on every failure signal
@task_failure.connect
def alert_on_failure(sender, task_id, exception, **kw):
    send_slack_message(f"🚨 Task {sender.name} failed: {exception}")
    
# If a task retries 3 times before succeeding, you get 3 Slack
# messages for something that ultimately WORKED.
# 100 tasks × 3 retries = 300 Slack messages per hour.
# Your team mutes the channel. Now real alerts are invisible.
# This is called ALERT FATIGUE.

# ✅ CORRECT: Alert only on FINAL failure (using on_failure hook)
class AlertingTask(Task):
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        # This only fires once: after all retries are exhausted.
        send_slack_message(
            f"🚨 DEAD LETTER: {self.name} [{task_id}] "
            f"failed permanently: {exc}"
        )
```

---

### Mistake 4: Using the schedule state file in Docker without a volume

```bash
# ❌ WRONG: Beat in Docker without persistent storage
# Beat creates celerybeat-schedule inside the container.
# When the container restarts, the file is gone.
# Beat thinks it has never run → fires ALL tasks immediately.

# ✅ CORRECT: Mount the schedule file to a volume
# In your docker-compose.yml (preview — Week 15):
# volumes:
#   - beat-schedule:/app/celerybeat-schedule
```

---

### Mistake 5: Forgetting timezone on crontab

```python
# ❌ WRONG: No timezone set
celery_app.conf.beat_schedule = {
    "daily-report": {
        "task": "tasks.generate_daily_report",
        "schedule": crontab(minute=0, hour=0),  # Midnight... which timezone?
    },
}
# On your laptop: midnight EST. On production: midnight UTC.
# That's a 5-hour difference.

# ✅ CORRECT: Explicit timezone
celery_app.conf.timezone = "UTC"
# Now crontab(minute=0, hour=0) ALWAYS means midnight UTC.
# Predictable. Deployable. Debuggable.
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│            SCHEDULING & MONITORING QUICK REFERENCE              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CELERY BEAT CONFIG:                                            │
│      celery_app.conf.beat_schedule = {                          │
│          "name": {                                              │
│              "task": "tasks.my_task",                           │
│              "schedule": crontab(minute=0, hour="*/3"),         │
│          },                                                     │
│      }                                                          │
│      celery_app.conf.timezone = "UTC"                           │
│                                                                 │
│  SCHEDULE TYPES:                                                │
│      timedelta(minutes=5)            Every 5 minutes            │
│      crontab(minute=0)              Every hour at :00           │
│      crontab(minute=0, hour=0)      Daily at midnight           │
│      crontab(minute=0, hour=9,      Mondays at 9 AM            │
│              day_of_week="mon")                                 │
│      crontab(minute="*/15")         Every 15 minutes            │
│                                                                 │
│  RUN BEAT:                                                      │
│      celery -A celery_app beat --loglevel=info                  │
│      ⚠️  ONE Beat instance only. Ever.                          │
│                                                                 │
│  TIMEOUTS:                                                      │
│      @celery_app.task(soft_time_limit=120, time_limit=150)      │
│      soft → raises SoftTimeLimitExceeded (catchable)            │
│      hard → SIGKILL (not catchable, process killed)             │
│                                                                 │
│  DEAD LETTERS:                                                  │
│      class AlertingTask(Task):                                  │
│          def on_failure(self, exc, task_id, args, kwargs, ...): │
│              # Store in DB, send alert                          │
│                                                                 │
│  FLOWER:                                                        │
│      pip install flower                                         │
│      celery -A celery_app flower --port=5555                    │
│      celery -A celery_app worker -E  (enable events)            │
│                                                                 │
│  SIGNALS:                                                       │
│      @task_failure.connect     → log all failures               │
│      @task_retry.connect       → track retry rates              │
│      @worker_ready.connect     → deployment verification        │
│                                                                 │
│  HEALTH CHECK:                                                  │
│      inspector = celery_app.control.inspect()                   │
│      inspector.ping()          → which workers are alive        │
│      inspector.active()        → what's running now             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  RELIABLE BACKGROUND SYSTEMS = SCHEDULE + PROTECT +             │
│                                OBSERVE + REACT                  │
│                                                                 │
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │ SCHEDULE │   │ PROTECT  │   │ OBSERVE  │   │  REACT   │    │
│  │          │   │          │   │          │   │          │    │
│  │  Celery  │──▶│ Timeouts │──▶│  Flower  │──▶│  Alerts  │    │
│  │   Beat   │   │ soft/hard│   │ + Health │   │ Signals  │    │
│  │          │   │          │   │  Checks  │   │ on_fail  │    │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘    │
│                                                                 │
│       "When"        "Stop it       "See it"      "Tell me"     │
│                    if stuck"                                    │
│                                                                 │
│                                                                 │
│  THE FACTORY FLOOR ANALOGY:                                     │
│  ├─ Beat = Production schedule board read by the shift manager  │
│  ├─ soft_time_limit = Warning buzzer on the machine             │
│  ├─ time_limit = Emergency power cutoff                         │
│  ├─ Dead letters = Reject bin for parts that always fail QC     │
│  ├─ Flower = Control room with screens showing every machine    │
│  └─ Signals + on_failure = Alarm system that pages the manager  │
│                                                                 │
│                                                                 │
│  THE RULE:                                                      │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  If nobody is watching, nobody knows it broke.         │     │
│  │  If nobody is alerted, nobody fixes it.                │     │
│  │  Build systems that report their own health.           │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Upcoming Content

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 11, LECTURE 4 (Next):                                     │
│  └─ Event-Driven Architecture Introduction                      │
│     Redis Pub/Sub for broadcasting events.                      │
│     When to reach for Kafka vs Celery vs BackgroundTasks.       │
│     Your dead letter handling connects to event sourcing        │
│     concepts — the idea that you record what happened.          │
│                                                                 │
│  WEEK 11 PROJECT:                                               │
│  └─ Background Processing Pipeline                              │
│     You'll USE Beat for scheduled jobs, timeouts to protect     │
│     workers, Flower to verify everything works, and alerting    │
│     hooks to catch failures. This lecture gives you the tools.  │
│                                                                 │
│  WEEK 12 (Performance):                                         │
│  └─ Load testing with locust                                    │
│     The health check endpoints you built today become           │
│     part of your load testing strategy — verify background      │
│     system health under load.                                   │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ Your SaaS platform will have scheduled tasks:               │
│     Report generation, data cleanup, notification digests.      │
│     Everything from today goes into production architecture.    │
│                                                                 │
│  WEEK 15 (Docker & CI/CD):                                      │
│  └─ Beat, workers, and Flower each become their own service     │
│     in Docker Compose. The single-Beat rule becomes a           │
│     deployment architecture decision.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```