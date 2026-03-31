# How to Use Testcontainers with Redis in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Python, Testcontainers, Integration Test, Docker

Description: Run real Redis integration tests in Python using Testcontainers to spin up a Redis Docker container automatically in your test suite with pytest fixtures.

---

Testcontainers starts a real Redis Docker container for each test run, giving you high-fidelity integration tests without a permanently running server. The container is created before tests and destroyed after, making tests fully isolated and CI-safe.

## Installation

```bash
pip install testcontainers[redis] pytest redis
```

## Basic Setup with pytest

```python
import pytest
import redis
from testcontainers.redis import RedisContainer

@pytest.fixture(scope="session")
def redis_container():
    with RedisContainer("redis:7-alpine") as container:
        yield container

@pytest.fixture(scope="session")
def redis_client(redis_container):
    client = redis.Redis(
        host=redis_container.get_container_host_ip(),
        port=redis_container.get_exposed_port(6379),
        decode_responses=True,
    )
    yield client
    client.close()

def test_set_and_get(redis_client):
    redis_client.set("hello", "world")
    assert redis_client.get("hello") == "world"

def test_expiry(redis_client):
    redis_client.set("temp", "value", ex=10)
    assert redis_client.ttl("temp") > 0

def test_hash_operations(redis_client):
    redis_client.hset("user:1", mapping={"name": "Alice", "role": "admin"})
    assert redis_client.hget("user:1", "name") == "Alice"
    assert redis_client.hget("user:1", "role") == "admin"
```

## Function-Scoped Container (Full Isolation)

For maximum test isolation, create a new container per test function. This is slower but ensures no state leaks between tests:

```python
@pytest.fixture(scope="function")
def isolated_redis():
    with RedisContainer("redis:7-alpine") as container:
        client = redis.Redis(
            host=container.get_container_host_ip(),
            port=container.get_exposed_port(6379),
            decode_responses=True,
        )
        yield client
        client.close()

def test_isolated_queue(isolated_redis):
    isolated_redis.rpush("jobs", "task1", "task2")
    assert isolated_redis.llen("jobs") == 2
    assert isolated_redis.lpop("jobs") == "task1"
```

## Testing Application Services

Inject the containerized Redis client into your service layer:

```python
from my_app.cache import UserCache

def test_user_cache_integration(redis_client):
    cache = UserCache(redis_client)

    # Set and retrieve
    cache.set_user(42, {"name": "Bob", "plan": "pro"}, ttl=60)
    user = cache.get_user(42)
    assert user["name"] == "Bob"
    assert user["plan"] == "pro"

    # TTL is set
    assert redis_client.ttl("user:42") > 0

    # Cache miss
    assert cache.get_user(9999) is None
```

## Testing Redis Streams

```python
def test_stream_producer_consumer(redis_client):
    stream_key = "test:events"

    # Produce
    redis_client.xadd(stream_key, {"type": "signup", "user_id": "1"})
    redis_client.xadd(stream_key, {"type": "login", "user_id": "2"})

    # Consume
    messages = redis_client.xrange(stream_key, "-", "+")
    assert len(messages) == 2
    assert messages[0][1]["type"] == "signup"
```

## Using with Docker Compose in CI

In CI environments with Docker-in-Docker, Testcontainers works by default. Set the `DOCKER_HOST` environment variable if needed:

```bash
export DOCKER_HOST=unix:///var/run/docker.sock
pytest tests/integration/
```

## Cleanup

Testcontainers automatically removes the container when the context manager exits. Use the `scope="session"` fixture for test suites that share a container to reduce startup overhead.

## Summary

Testcontainers for Python spins up a real Redis Docker container in your test suite using a pytest fixture. Use `scope="session"` for performance across a test suite or `scope="function"` for full test isolation. This gives you accurate integration tests against real Redis behavior without a permanent infrastructure dependency.
