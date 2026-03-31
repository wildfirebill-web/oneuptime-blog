# How to Set Up Redis Test Services in CI/CD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, CI/CD, Testing, Service Container, Docker

Description: Learn best practices for setting up Redis test services in CI/CD pipelines, including isolation, health checks, seed data, and cleanup strategies.

---

Running Redis in CI/CD requires more than just starting a container. You need isolation between test runs, health checks to prevent flaky failures, and proper cleanup. This guide covers the patterns that make Redis CI services reliable.

## The Core Problem: Flaky Redis in CI

The most common CI failures with Redis are:
1. Tests start before Redis is ready
2. Test data from one run contaminates the next
3. Multiple parallel jobs sharing the same Redis instance

## Pattern 1: Health Checks Before Tests

Never assume Redis is ready immediately after the container starts. Always verify:

```bash
#!/bin/bash
# wait-for-redis.sh
MAX_RETRIES=30
RETRY_INTERVAL=2
HOST=${REDIS_HOST:-localhost}
PORT=${REDIS_PORT:-6379}

for i in $(seq 1 $MAX_RETRIES); do
    if redis-cli -h "$HOST" -p "$PORT" ping 2>/dev/null | grep -q PONG; then
        echo "Redis is ready after $i attempts"
        exit 0
    fi
    echo "Attempt $i/$MAX_RETRIES: Redis not ready, waiting ${RETRY_INTERVAL}s..."
    sleep $RETRY_INTERVAL
done

echo "Redis failed to start after $((MAX_RETRIES * RETRY_INTERVAL)) seconds"
exit 1
```

Use it in any CI system:

```yaml
# GitHub Actions
- name: Wait for Redis
  run: ./scripts/wait-for-redis.sh
  env:
    REDIS_HOST: localhost
```

## Pattern 2: Database Isolation

Use separate Redis databases (0-15) per test suite to avoid contamination:

```python
# conftest.py
import redis
import pytest
import os

@pytest.fixture(scope="session")
def redis_client():
    # Use database 1 for integration tests, 0 for default
    r = redis.Redis(
        host=os.getenv("REDIS_HOST", "localhost"),
        port=int(os.getenv("REDIS_PORT", 6379)),
        db=int(os.getenv("REDIS_TEST_DB", 1)),
        decode_responses=True
    )
    yield r
    # Flush only the test database after the session
    r.flushdb()
```

## Pattern 3: Key Namespacing

Prefix all test keys to make cleanup easy and avoid accidental collisions:

```python
class TestRedisCache:
    PREFIX = "test:cache:"

    def setup_method(self):
        self.redis = redis.Redis(host="localhost", decode_responses=True)

    def teardown_method(self):
        # Clean up only keys created by this test class
        keys = self.redis.keys(f"{self.PREFIX}*")
        if keys:
            self.redis.delete(*keys)

    def test_set_and_get(self):
        key = f"{self.PREFIX}user:1"
        self.redis.set(key, "John Doe", ex=60)
        assert self.redis.get(key) == "John Doe"
```

## Pattern 4: Seed Data Fixture

Load test fixtures into Redis before the test suite runs:

```bash
#!/bin/bash
# seed-redis.sh
REDIS_CLI="redis-cli -h ${REDIS_HOST:-localhost} -p ${REDIS_PORT:-6379}"

echo "Seeding Redis test data..."
$REDIS_CLI set "config:feature_flags" '{"new_checkout":true,"beta_ui":false}'
$REDIS_CLI set "config:app_settings" '{"timeout":30,"max_retries":3}'
$REDIS_CLI sadd "users:active" "user:1" "user:2" "user:3"
echo "Seed complete"
```

Call it in your pipeline:

```yaml
# .gitlab-ci.yml
before_script:
  - ./scripts/wait-for-redis.sh
  - ./scripts/seed-redis.sh
```

## Pattern 5: Docker Compose for Multi-Service Tests

When Redis is one of many services, Docker Compose keeps things clean:

```yaml
# docker-compose.test.yml
services:
  redis:
    image: redis:7.2-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 3s
      timeout: 2s
      retries: 10
    tmpfs:
      - /data  # Use in-memory storage for speed in CI
```

Use `tmpfs` for the data directory in CI to make Redis faster and avoid PVC issues.

## Summary

Reliable Redis in CI/CD requires three things: a health check loop that waits for the container before running tests, database isolation or key namespacing to prevent cross-test contamination, and a cleanup step that runs even when tests fail. These patterns together eliminate the flakiness that plagues most Redis CI setups.
