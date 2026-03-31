# How to Run Redis Integration Tests in CI/CD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Integration Test, CI/CD, Testing, Python

Description: Learn how to write and run Redis integration tests in CI/CD pipelines, covering test structure, fixtures, assertions, and cleanup strategies.

---

Integration tests that use Redis verify that your application's caching, rate limiting, and pub/sub logic work correctly against a real Redis instance. This guide covers writing effective Redis integration tests and running them reliably in CI/CD.

## What to Test in Redis Integration Tests

Integration tests should cover:
- Cache set/get/expire behavior
- Atomic operations (INCR, SETNX)
- TTL and expiration
- Pub/sub message delivery
- Pipeline and transaction correctness

## Python: pytest with Redis Fixtures

```python
# tests/conftest.py
import redis
import pytest
import os

@pytest.fixture(scope="session")
def redis_url():
    return os.getenv("REDIS_URL", "redis://localhost:6379/1")

@pytest.fixture(scope="session")
def r(redis_url):
    client = redis.from_url(redis_url, decode_responses=True)
    client.ping()
    yield client
    client.flushdb()

@pytest.fixture(autouse=True)
def cleanup_keys(r, request):
    yield
    # Clean up keys created during the test
    pattern = f"test:{request.node.name}:*"
    keys = r.keys(pattern)
    if keys:
        r.delete(*keys)
```

Integration tests:

```python
# tests/integration/test_cache.py
import time

def test_cache_stores_and_retrieves(r):
    r.set("test:set_get:key", "hello", ex=60)
    assert r.get("test:set_get:key") == "hello"

def test_cache_ttl_expires(r):
    r.set("test:ttl:key", "value", ex=1)
    assert r.get("test:ttl:key") == "value"
    time.sleep(1.5)
    assert r.get("test:ttl:key") is None

def test_rate_limiter_increments(r):
    key = "test:rate:user:1"
    r.delete(key)
    r.incr(key)
    r.incr(key)
    r.incr(key)
    count = r.get(key)
    assert int(count) == 3

def test_setnx_idempotent_lock(r):
    key = "test:lock:resource_1"
    r.delete(key)
    result1 = r.setnx(key, "locked")
    result2 = r.setnx(key, "locked-again")
    assert result1 is True
    assert result2 is False
    assert r.get(key) == "locked"
```

## Node.js: Jest with ioredis

```javascript
// tests/integration/redis.test.js
const Redis = require('ioredis');

let redis;

beforeAll(async () => {
  redis = new Redis({
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379'),
    db: 1
  });
  await redis.ping();
});

afterAll(async () => {
  await redis.flushdb();
  await redis.quit();
});

test('should store and retrieve a value', async () => {
  await redis.set('test:user:1', JSON.stringify({ name: 'Alice' }), 'EX', 60);
  const value = await redis.get('test:user:1');
  expect(JSON.parse(value)).toEqual({ name: 'Alice' });
});

test('should expire keys correctly', async () => {
  await redis.set('test:expiry:key', 'value', 'PX', 500); // 500ms
  const immediate = await redis.get('test:expiry:key');
  expect(immediate).toBe('value');
  await new Promise(r => setTimeout(r, 600));
  const expired = await redis.get('test:expiry:key');
  expect(expired).toBeNull();
});
```

## CI/CD Pipeline Configuration

GitHub Actions example with test isolation:

```yaml
name: Integration Tests

on: [push, pull_request]

jobs:
  integration:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis:7.2-alpine
        ports:
        - 6379:6379
        options: --health-cmd "redis-cli ping" --health-interval 5s --health-timeout 3s --health-retries 10
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-python@v5
      with:
        python-version: '3.12'
    - run: pip install redis pytest
    - run: pytest tests/integration/ -v --tb=short
      env:
        REDIS_URL: redis://localhost:6379/1
```

## Summary

Redis integration tests should cover cache operations, TTL behavior, atomic operations, and cleanup. Use a dedicated Redis database (db 1 or higher) for tests to isolate from development data, implement autouse fixtures for automatic cleanup, and configure health checks in CI to guarantee Redis is ready before tests begin. These practices make your Redis integration test suite fast and reliable.
