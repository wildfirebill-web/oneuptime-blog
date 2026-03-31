# How to Mock Redis in Unit Tests (Best Practices)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Unit Tests, Mocking, Python, Node.js, Testing Best Practices

Description: Mock Redis in unit tests using fakeredis, ioredis-mock, and miniredis to test application logic without a running Redis server for fast and reliable unit tests.

---

## When to Mock vs. Use Real Redis

Unit tests should test application logic in isolation - not Redis behavior. Use mocks for:
- Testing business logic that happens to use Redis
- CI environments without Docker
- Tests that need to run in milliseconds

Use real Redis (via Testcontainers or Docker) for:
- Integration tests validating actual Redis behavior
- Tests for TTL/expiry, pub/sub, scripting, or cluster behavior

## Python - fakeredis

`fakeredis` is the standard in-memory Redis mock for Python:

```bash
pip install fakeredis
```

### Basic Usage

```python
import fakeredis
import redis

def test_with_fakeredis():
    r = fakeredis.FakeRedis(decode_responses=True)
    r.set("key", "value")
    assert r.get("key") == "value"
    r.rpush("list", "a", "b", "c")
    assert r.llen("list") == 3
```

### Testing Application Code

```python
import fakeredis
import pytest

class UserCache:
    def __init__(self, redis_client):
        self.r = redis_client

    def cache_user(self, user_id: str, data: dict, ttl: int = 3600):
        import json
        self.r.set(f"user:{user_id}", json.dumps(data), ex=ttl)

    def get_user(self, user_id: str):
        import json
        val = self.r.get(f"user:{user_id}")
        return json.loads(val) if val else None

    def invalidate_user(self, user_id: str):
        self.r.delete(f"user:{user_id}")

@pytest.fixture
def cache():
    fake_redis = fakeredis.FakeRedis(decode_responses=True)
    return UserCache(fake_redis)

def test_cache_and_retrieve(cache):
    cache.cache_user("101", {"name": "Alice", "role": "admin"})
    result = cache.get_user("101")
    assert result["name"] == "Alice"

def test_invalidation(cache):
    cache.cache_user("101", {"name": "Alice"})
    cache.invalidate_user("101")
    assert cache.get_user("101") is None

def test_miss_returns_none(cache):
    assert cache.get_user("999") is None
```

### Async fakeredis

```python
import asyncio
import fakeredis.aioredis

async def test_async_redis():
    r = fakeredis.aioredis.FakeRedis(decode_responses=True)
    await r.set("key", "async_value")
    val = await r.get("key")
    assert val == "async_value"
```

## Node.js - ioredis-mock

```bash
npm install --save-dev ioredis-mock
```

```javascript
const RedisMock = require("ioredis-mock");

jest.mock("ioredis", () => require("ioredis-mock"));

class SessionStore {
  constructor(redisClient) {
    this.redis = redisClient;
  }

  async setSession(token, userId, ttl = 3600) {
    await this.redis.set(`session:${token}`, userId, "EX", ttl);
  }

  async getSession(token) {
    return await this.redis.get(`session:${token}`);
  }

  async deleteSession(token) {
    await this.redis.del(`session:${token}`);
  }
}

describe("SessionStore", () => {
  let store;

  beforeEach(() => {
    store = new SessionStore(new RedisMock());
  });

  test("stores and retrieves session", async () => {
    await store.setSession("tok123", "user:101");
    const userId = await store.getSession("tok123");
    expect(userId).toBe("user:101");
  });

  test("returns null for missing session", async () => {
    const result = await store.getSession("missing");
    expect(result).toBeNull();
  });

  test("deletes session", async () => {
    await store.setSession("tok123", "user:101");
    await store.deleteSession("tok123");
    const result = await store.getSession("tok123");
    expect(result).toBeNull();
  });
});
```

## Go - miniredis

```bash
go get github.com/alicebob/miniredis/v2
```

```go
package cache_test

import (
    "testing"
    "time"

    "github.com/alicebob/miniredis/v2"
    "github.com/redis/go-redis/v9"
    "context"
)

func setupTestRedis(t *testing.T) (*miniredis.Miniredis, *redis.Client) {
    mr := miniredis.RunT(t)
    client := redis.NewClient(&redis.Options{
        Addr: mr.Addr(),
    })
    return mr, client
}

func TestSetAndGet(t *testing.T) {
    _, client := setupTestRedis(t)
    ctx := context.Background()

    client.Set(ctx, "key", "value", 0)
    val, _ := client.Get(ctx, "key").Result()
    if val != "value" {
        t.Errorf("expected value, got %s", val)
    }
}

func TestExpiry(t *testing.T) {
    mr, client := setupTestRedis(t)
    ctx := context.Background()

    client.Set(ctx, "temp", "data", time.Second)
    mr.FastForward(2 * time.Second)

    _, err := client.Get(ctx, "temp").Result()
    if err != redis.Nil {
        t.Error("expected key to be expired")
    }
}
```

## Best Practice - Dependency Injection

Always inject the Redis client rather than instantiating it inside your classes. This makes mocking seamless:

```python
# Good - injectable
class RateLimiter:
    def __init__(self, redis_client):
        self.r = redis_client

# Bad - hard to mock
class RateLimiter:
    def __init__(self):
        import redis
        self.r = redis.Redis()
```

## Summary

Mocking Redis in unit tests isolates application logic from infrastructure. Python's `fakeredis`, Node.js's `ioredis-mock`, and Go's `miniredis` provide in-memory Redis implementations that support the full command API. Always design application code with dependency injection to make substituting a mock Redis client effortless in tests.
