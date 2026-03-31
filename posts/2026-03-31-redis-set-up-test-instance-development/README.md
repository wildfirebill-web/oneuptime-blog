# How to Set Up a Test Redis Instance for Development

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Testing, Development, Docker, Configuration

Description: Set up a dedicated Redis test instance for local development using Docker, with isolation strategies and configuration tuning for fast test cycles.

---

Running tests against your production Redis instance is risky - tests can corrupt data, cause flapping from leftover keys, and slow down production traffic. The simplest way to get a safe, fast Redis instance for development and testing is Docker.

## Quick Start with Docker

```bash
# Start a Redis instance on a non-default port for test isolation
docker run --name redis-test \
  -p 6380:6379 \
  -d redis:7 \
  redis-server --save "" --appendonly no --loglevel warning
```

Key flags:
- `--save ""` disables RDB persistence (tests run faster).
- `--appendonly no` disables AOF (no disk writes during tests).
- Port 6380 avoids conflicts with a dev instance on 6379.

## Connecting Your Tests to the Test Instance

```python
import redis
import os

def get_test_redis():
    host = os.getenv("REDIS_TEST_HOST", "localhost")
    port = int(os.getenv("REDIS_TEST_PORT", 6380))
    return redis.Redis(host=host, port=port, decode_responses=True)
```

Set environment variables in your test runner or CI config rather than hardcoding.

## Flushing Between Tests

Each test should start with a clean database. Flush the database at the start of each test or test suite:

```python
import pytest
import redis

@pytest.fixture(autouse=True)
def clean_redis():
    r = redis.Redis(host="localhost", port=6380, decode_responses=True)
    r.flushdb()
    yield r
    r.flushdb()
```

For parallel test suites, use separate Redis databases (0-15) per worker instead of FLUSHDB:

```python
@pytest.fixture
def redis_client(worker_id):
    db = int(worker_id.replace("gw", "")) if worker_id != "master" else 0
    r = redis.Redis(host="localhost", port=6380, db=db, decode_responses=True)
    r.flushdb()
    yield r
```

## Docker Compose for a Full Dev Stack

```yaml
version: "3.9"
services:
  redis-dev:
    image: redis:7
    ports:
      - "6379:6379"
    volumes:
      - redis-dev-data:/data

  redis-test:
    image: redis:7
    ports:
      - "6380:6379"
    command: redis-server --save "" --appendonly no

volumes:
  redis-dev-data:
```

Run both with `docker compose up -d` and point dev code at 6379 and tests at 6380.

## Configuring Memory Limits for Tests

Prevent tests from consuming excessive memory by setting a limit:

```bash
docker run --name redis-test \
  -p 6380:6379 \
  -d redis:7 \
  redis-server --save "" --appendonly no --maxmemory 256mb --maxmemory-policy allkeys-lru
```

## Resetting Configuration at Runtime

If you need to test different Redis configuration scenarios, change settings at runtime without restarting:

```bash
# Connect to test instance
redis-cli -p 6380

# Change config for a specific test
CONFIG SET hz 100
CONFIG SET maxmemory-policy volatile-lru
```

## Summary

Use a separate Docker-based Redis instance on a non-default port for testing, with persistence disabled for speed. Flush the database before and after each test using a pytest fixture or equivalent. For parallel test runners, assign separate Redis databases per worker to keep tests fully isolated without restarts.
