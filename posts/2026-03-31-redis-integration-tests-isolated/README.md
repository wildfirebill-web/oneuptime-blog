# How to Write Redis Integration Tests That Are Isolated

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Integration Tests, Test Isolation, pytest, Jest, Testing

Description: Write Redis integration tests with full isolation using FLUSHDB, database selection, key prefixes, and test fixtures to prevent cross-test contamination.

---

## Why Test Isolation Matters

Integration tests against a shared Redis instance often fail intermittently because one test's data leaks into another. Proper isolation ensures tests are independent, repeatable, and can run in any order - whether locally or in CI.

## Strategy 1 - FLUSHDB Between Tests

The simplest approach: flush the entire database before or after each test.

```python
import redis
import pytest

@pytest.fixture(scope="session")
def redis_client():
    return redis.Redis(host="localhost", port=6379, db=15, decode_responses=True)

@pytest.fixture(autouse=True)
def clean_redis(redis_client):
    yield
    redis_client.flushdb()

def test_set_value(redis_client):
    redis_client.set("foo", "bar")
    assert redis_client.get("foo") == "bar"

def test_no_leftover_from_previous_test(redis_client):
    assert redis_client.get("foo") is None
```

Use `db=15` for tests to avoid colliding with application data in `db=0`.

## Strategy 2 - Per-Test Key Prefixes

Generate unique prefixes per test run to avoid conflicts without flushing:

```python
import uuid
import redis
import pytest

@pytest.fixture
def test_prefix():
    return f"test:{uuid.uuid4().hex[:8]}:"

@pytest.fixture
def redis_client():
    return redis.Redis(decode_responses=True)

def test_user_cache(redis_client, test_prefix):
    key = f"{test_prefix}user:101"
    redis_client.set(key, "Alice", ex=60)
    assert redis_client.get(key) == "Alice"

def test_counter(redis_client, test_prefix):
    key = f"{test_prefix}counter"
    redis_client.incr(key)
    redis_client.incr(key)
    assert redis_client.get(key) == "2"
```

This approach works well when you cannot afford to flush a shared database.

## Strategy 3 - Separate Redis Databases

Redis has 16 databases (0-15). Assign each test module a different database:

```python
# conftest.py
import redis
import pytest

@pytest.fixture(scope="module")
def user_tests_redis():
    r = redis.Redis(db=1, decode_responses=True)
    yield r
    r.flushdb()

@pytest.fixture(scope="module")
def session_tests_redis():
    r = redis.Redis(db=2, decode_responses=True)
    yield r
    r.flushdb()
```

## Strategy 4 - Testcontainers for Full Isolation

Each test class or module gets its own Redis container:

```python
from testcontainers.redis import RedisContainer
import redis
import pytest

@pytest.fixture(scope="class")
def isolated_redis():
    with RedisContainer("redis:7-alpine") as container:
        client = redis.Redis(
            host=container.get_container_host_ip(),
            port=container.get_exposed_port(6379),
            decode_responses=True
        )
        yield client

class TestUserService:
    def test_set_user(self, isolated_redis):
        isolated_redis.hset("user:1", mapping={"name": "Alice"})
        assert isolated_redis.hget("user:1", "name") == "Alice"

    def test_delete_user(self, isolated_redis):
        isolated_redis.hset("user:2", mapping={"name": "Bob"})
        isolated_redis.delete("user:2")
        assert isolated_redis.hgetall("user:2") == {}
```

## Node.js - Jest with ioredis-mock

Use a fresh mock instance per test to guarantee isolation:

```javascript
const RedisMock = require("ioredis-mock");

describe("CartService", () => {
  let redis;
  let cart;

  beforeEach(() => {
    redis = new RedisMock();
    cart = new CartService(redis);
  });

  test("adds item to cart", async () => {
    await cart.addItem("user:1", "product:101", 2);
    const count = await redis.hget("cart:user:1", "product:101");
    expect(count).toBe("2");
  });

  test("removes item from cart", async () => {
    await cart.addItem("user:1", "product:101", 1);
    await cart.removeItem("user:1", "product:101");
    const count = await redis.hget("cart:user:1", "product:101");
    expect(count).toBeNull();
  });
});
```

## Java - Test Isolation with @BeforeEach

```java
@SpringBootTest
class RedisServiceTest {

    @Autowired
    private StringRedisTemplate redisTemplate;

    @BeforeEach
    void flushTestDb() {
        redisTemplate.getConnectionFactory()
            .getConnection()
            .flushDb();
    }

    @Test
    void testStoreAndRetrieve() {
        redisTemplate.opsForValue().set("key", "value");
        assertEquals("value", redisTemplate.opsForValue().get("key"));
    }
}
```

## Cleanup Tracker Pattern

If you cannot flush the entire DB, track keys created by each test and clean them up:

```python
class RedisTestTracker:
    def __init__(self, redis_client):
        self.r = redis_client
        self._keys = []

    def set(self, key, value, **kwargs):
        self._keys.append(key)
        return self.r.set(key, value, **kwargs)

    def hset(self, key, *args, **kwargs):
        self._keys.append(key)
        return self.r.hset(key, *args, **kwargs)

    def cleanup(self):
        if self._keys:
            self.r.delete(*self._keys)
            self._keys.clear()

@pytest.fixture
def tracked_redis():
    r = redis.Redis(decode_responses=True)
    tracker = RedisTestTracker(r)
    yield tracker
    tracker.cleanup()
```

## Summary

Redis integration test isolation requires either flushing the database between tests (simplest), using unique key prefixes per test, partitioning tests across Redis databases, or spinning up dedicated containers per test class. FLUSHDB in an autouse fixture combined with a dedicated test database number is the recommended default approach for most projects.
