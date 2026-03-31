# How to Use Redis with Celery Beat for Periodic Tasks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Celery, Task Queue

Description: Learn how to configure Celery Beat with Redis as both the broker and result backend, including schedule management, task deduplication, and monitoring.

---

Celery Beat is the periodic task scheduler for Celery. With Redis as the broker and result backend, you get a complete task scheduling system without a separate message broker infrastructure.

## Setup: Celery with Redis Broker and Backend

```python
# celery_app.py
from celery import Celery

app = Celery(
    "myapp",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/1",
    include=["myapp.tasks"],
)

app.conf.update(
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],
    timezone="UTC",
    enable_utc=True,
    result_expires=3600,  # Results expire after 1 hour
    task_acks_late=True,   # Acknowledge after completion, not on receipt
    task_reject_on_worker_lost=True,
)
```

## Define Periodic Tasks with Beat Schedule

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    "cleanup-expired-sessions": {
        "task": "myapp.tasks.cleanup_sessions",
        "schedule": crontab(hour=2, minute=0),  # Daily at 2 AM
        "args": (),
    },
    "refresh-analytics-cache": {
        "task": "myapp.tasks.refresh_analytics",
        "schedule": 300.0,  # Every 5 minutes
        "kwargs": {"force": False},
    },
    "send-digest-emails": {
        "task": "myapp.tasks.send_digests",
        "schedule": crontab(hour=8, minute=0, day_of_week="mon-fri"),
        "args": ("daily",),
    },
}
```

## Define the Tasks

```python
# tasks.py
from celery_app import app
import redis
import json

cache = redis.Redis(host="localhost", port=6379, decode_responses=True)

@app.task(
    name="myapp.tasks.refresh_analytics",
    bind=True,
    max_retries=3,
    default_retry_delay=60,
)
def refresh_analytics(self, force: bool = False):
    cache_key = "analytics:dashboard:summary"

    if not force and cache.exists(cache_key):
        return {"status": "skipped", "reason": "cache_hit"}

    try:
        data = compute_analytics_summary()
        cache.set(cache_key, json.dumps(data), ex=600)
        return {"status": "refreshed", "keys": list(data.keys())}
    except Exception as exc:
        self.retry(exc=exc)
```

## Prevent Duplicate Concurrent Executions

Use Redis locks to ensure only one instance of a task runs at a time:

```python
@app.task(name="myapp.tasks.cleanup_sessions")
def cleanup_sessions():
    lock_key = "celery:lock:cleanup_sessions"
    lock = cache.set(lock_key, "1", nx=True, ex=120)  # Max 2 min lock

    if not lock:
        return {"status": "skipped", "reason": "already_running"}

    try:
        count = delete_expired_sessions()
        return {"status": "done", "deleted": count}
    finally:
        cache.delete(lock_key)
```

## Start Beat and Worker

```bash
# Start the scheduler
celery -A celery_app beat --loglevel=info

# Start a worker in a separate terminal
celery -A celery_app worker --loglevel=info --concurrency=4
```

## Monitor Task Execution

```python
# Check how many tasks are in the queue
def queue_depth(queue_name: str = "celery") -> int:
    r = redis.Redis(host="localhost", port=6379)
    return r.llen(queue_name)

# Check result of a specific task
from celery_app import app

def get_task_status(task_id: str) -> dict:
    result = app.AsyncResult(task_id)
    return {
        "id": task_id,
        "state": result.state,
        "result": result.result if result.ready() else None,
    }
```

## Summary

Celery Beat with Redis provides a complete periodic task system that requires only Redis as infrastructure. Define schedules in `beat_schedule`, use task-level Redis locks to prevent duplicate concurrent runs of critical jobs, and monitor queue depth and task states directly through Redis to detect backlogs or failures before they impact users.
