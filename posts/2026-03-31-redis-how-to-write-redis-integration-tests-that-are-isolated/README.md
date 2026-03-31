# How to Write Redis Integration Tests That Are Isolated

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Integration Testing, Test Isolation, Pytest, Best Practices

Description: Learn how to write isolated Redis integration tests using database selection, key prefixing, and per-test flush strategies to prevent state leakage.

---

## The Problem with Non-Isolated Redis Tests

Integration tests that share a Redis instance without proper isolation cause flaky tests. A test that expects an empty sorted set will fail if a previous test left data behind. Shared counters, cached values, and pub/sub subscriptions from other tests can corrupt results silently.

## Strategy 1: Use Separate Redis Databases

Redis supports 16 databases (0-15) by default. Assign a dedicated database to your test suite:

```python
import redis
import pytest

TEST_DB = 15

@pytest.fixture(scope="session")
def redis_client():
    r = redis.Redis(host='localhost', port=6379, db=TEST_DB, decode_responses=True)
    yield r
    r.flushdb()  # Clean up after entire session
    r.close()

@pytest.fixture(autouse=True)
def clean_db(redis_client):
    redis_client.flushdb()
    yield
    # Optional: flush again after test
```

The `autouse=True` fixture runs before every test automatically, giving each test a clean slate.

## Strategy 2: Key Prefix Namespacing

For environments where you cannot isolate by database, use a unique key prefix per test:

```python
import pytest
import redis
import uuid

@pytest.fixture
def test_prefix():
    return f"test:{uuid.uuid4().hex[:8]}:"

@pytest.fixture
def redis_client():
    r = redis.Redis(host='localhost', port=6379, decode_responses=True)
    yield r
    r.close()

@pytest.fixture
def scoped_redis(redis_client, test_prefix):
    """A Redis client that auto-prefixes all keys"""
    class ScopedRedis:
        def __init__(self, client, prefix):
            self._r = client
            self._prefix = prefix

        def set(self, key, value, **kwargs):
            return self._r.set(self._prefix + key, value, **kwargs)

        def get(self, key):
            return self._r.get(self._prefix + key)

        def hset(self, key, mapping=None, **kwargs):
            return self._r.hset(self._prefix + key, mapping=mapping, **kwargs)

        def hgetall(self, key):
            return self._r.hgetall(self._prefix + key)

        def delete(self, *keys):
            return self._r.delete(*[self._prefix + k for k in keys])

        def cleanup(self):
            pattern = self._prefix + "*"
            keys = self._r.keys(pattern)
            if keys:
                self._r.delete(*keys)

    scoped = ScopedRedis(redis_client, test_prefix)
    yield scoped
    scoped.cleanup()

def test_user_cache(scoped_redis):
    scoped_redis.set("user:1", "Alice")
    assert scoped_redis.get("user:1") == "Alice"

def test_user_hash(scoped_redis):
    scoped_redis.hset("user:1", mapping={"name": "Bob", "age": "25"})
    data = scoped_redis.hgetall("user:1")
    assert data["name"] == "Bob"
```

## Strategy 3: flushdb in Setup and Teardown

For test suites where you own the Redis instance:

```python
import pytest
import redis

@pytest.fixture(scope="module")
def r():
    client = redis.Redis(host='localhost', port=6379, db=15, decode_responses=True)
    client.flushdb()
    yield client
    client.flushdb()
    client.close()

class TestCacheOperations:
    def setup_method(self):
        self.r = redis.Redis(host='localhost', port=6379, db=15, decode_responses=True)
        self.r.flushdb()

    def teardown_method(self):
        self.r.flushdb()
        self.r.close()

    def test_counter_starts_at_zero(self):
        assert self.r.get("counter") is None
        self.r.incr("counter")
        assert self.r.get("counter") == "1"

    def test_set_with_ttl(self):
        import time
        self.r.setex("temp", 1, "value")
        assert self.r.get("temp") == "value"
        time.sleep(1.1)
        assert self.r.get("temp") is None
```

## Strategy 4: Testcontainers per Module

Start a fresh Redis container per test module for full isolation with no shared state risk:

```python
import pytest
import redis
from testcontainers.redis import RedisContainer

@pytest.fixture(scope="module")
def fresh_redis():
    with RedisContainer("redis:7-alpine") as container:
        client = redis.Redis(
            host=container.get_container_host_ip(),
            port=container.get_exposed_port(6379),
            decode_responses=True
        )
        yield client

def test_leaderboard_initial_state(fresh_redis):
    fresh_redis.zadd("leaderboard", {"alice": 100})
    rank = fresh_redis.zrevrank("leaderboard", "alice")
    assert rank == 0

def test_leaderboard_update(fresh_redis):
    fresh_redis.zadd("leaderboard", {"alice": 100, "bob": 200})
    top = fresh_redis.zrevrange("leaderboard", 0, 0)
    assert top == ["bob"]
```

## Isolating Pub/Sub Tests

Pub/Sub tests are particularly sensitive to cross-test contamination. Use unique channel names:

```python
import pytest
import redis
import threading
import uuid
import time

@pytest.fixture
def channel():
    return f"test-channel:{uuid.uuid4().hex[:8]}"

def test_pubsub_message_delivery(redis_client, channel):
    received_messages = []

    sub = redis_client.pubsub()
    sub.subscribe(channel)

    def listener():
        for msg in sub.listen():
            if msg['type'] == 'message':
                received_messages.append(msg['data'])
                break

    thread = threading.Thread(target=listener, daemon=True)
    thread.start()

    time.sleep(0.1)  # Allow subscription to settle
    redis_client.publish(channel, "hello")
    thread.join(timeout=2)

    assert received_messages == ["hello"]
    sub.close()
```

## Parallel Test Safety

When running tests in parallel (e.g., with pytest-xdist), use worker IDs for database assignment:

```python
# conftest.py
import pytest
import redis

def pytest_configure(config):
    # Each xdist worker gets its own Redis DB
    worker_id = getattr(config, 'workerinput', {}).get('workerid', 'master')
    db_map = {'master': 0, 'gw0': 1, 'gw1': 2, 'gw2': 3, 'gw3': 4}
    config.redis_db = db_map.get(worker_id, 15)

@pytest.fixture(scope="session")
def redis_client(pytestconfig):
    db = getattr(pytestconfig, 'redis_db', 15)
    r = redis.Redis(host='localhost', port=6379, db=db, decode_responses=True)
    r.flushdb()
    yield r
    r.flushdb()
```

## Summary

Writing isolated Redis integration tests requires a deliberate strategy: use separate database indices (db=15) for test suites, flush the database before each test with autouse fixtures, or use unique key prefixes when database isolation is not possible. For maximum isolation with zero state sharing, use Testcontainers to spin up a fresh Redis instance per module. Always clean up after each test to prevent flakiness caused by leftover keys, expired but still-visible data, or lingering pub/sub subscriptions.
