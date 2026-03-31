# How to Use Testcontainers for Redis Integration Tests

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Testcontainers, Integration Tests, Java, Python, Testing

Description: Use Testcontainers to spin up real Redis containers programmatically in Java, Python, and Go integration tests for fast, isolated, and reproducible test environments.

---

## What is Testcontainers

Testcontainers is a library that manages Docker containers programmatically from within test code. It starts a container before your test, provides the host/port, and shuts it down after. This eliminates manual Docker setup, works in CI pipelines, and ensures every test run gets a fresh Redis instance.

## Java with JUnit 5

Add dependency in `pom.xml`:

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.19.7</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
    <version>6.3.2.RELEASE</version>
</dependency>
```

Test class:

```java
import org.junit.jupiter.api.*;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.utility.DockerImageName;
import io.lettuce.core.RedisClient;
import io.lettuce.core.api.sync.RedisCommands;

@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class RedisIntegrationTest {

    static GenericContainer<?> redis = new GenericContainer<>(
            DockerImageName.parse("redis:7-alpine"))
        .withExposedPorts(6379);

    static RedisCommands<String, String> commands;

    @BeforeAll
    static void setUp() {
        redis.start();
        String url = "redis://" + redis.getHost() + ":" + redis.getMappedPort(6379);
        RedisClient client = RedisClient.create(url);
        commands = client.connect().sync();
    }

    @BeforeEach
    void flushAll() {
        commands.flushall();
    }

    @AfterAll
    static void tearDown() {
        redis.stop();
    }

    @Test
    @Order(1)
    void testSetAndGet() {
        commands.set("key", "value");
        Assertions.assertEquals("value", commands.get("key"));
    }

    @Test
    @Order(2)
    void testExpiry() throws InterruptedException {
        commands.setex("temp", 1, "data");
        Assertions.assertEquals("data", commands.get("temp"));
        Thread.sleep(1500);
        Assertions.assertNull(commands.get("temp"));
    }

    @Test
    @Order(3)
    void testHashOperations() {
        commands.hset("user:1", Map.of("name", "Alice", "role", "admin"));
        Assertions.assertEquals("Alice", commands.hget("user:1", "name"));
    }
}
```

## Python with pytest-testcontainers

Install dependencies:

```bash
pip install testcontainers redis pytest
```

Write the test:

```python
import pytest
import redis as redis_lib
from testcontainers.redis import RedisContainer

@pytest.fixture(scope="session")
def redis_client():
    with RedisContainer("redis:7-alpine") as container:
        client = redis_lib.Redis(
            host=container.get_container_host_ip(),
            port=container.get_exposed_port(6379),
            decode_responses=True
        )
        yield client

@pytest.fixture(autouse=True)
def flush(redis_client):
    redis_client.flushall()
    yield

def test_string_operations(redis_client):
    redis_client.set("hello", "world")
    assert redis_client.get("hello") == "world"

def test_counter(redis_client):
    redis_client.set("counter", 0)
    redis_client.incr("counter")
    redis_client.incr("counter")
    assert redis_client.get("counter") == "2"

def test_sorted_set(redis_client):
    redis_client.zadd("leaderboard", {"alice": 150, "bob": 200, "carol": 100})
    top = redis_client.zrevrange("leaderboard", 0, 0)
    assert top == ["bob"]
```

## Go with Testcontainers-Go

```go
package integration_test

import (
    "context"
    "testing"

    "github.com/redis/go-redis/v9"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/wait"
)

func setupRedis(t *testing.T) *redis.Client {
    ctx := context.Background()
    req := testcontainers.ContainerRequest{
        Image:        "redis:7-alpine",
        ExposedPorts: []string{"6379/tcp"},
        WaitingFor:   wait.ForLog("Ready to accept connections"),
    }
    container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: req,
        Started:          true,
    })
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() { container.Terminate(ctx) })

    host, _ := container.Host(ctx)
    port, _ := container.MappedPort(ctx, "6379")

    return redis.NewClient(&redis.Options{
        Addr: host + ":" + port.Port(),
    })
}

func TestRedisSet(t *testing.T) {
    rdb := setupRedis(t)
    ctx := context.Background()

    err := rdb.Set(ctx, "key", "value", 0).Err()
    if err != nil {
        t.Fatal(err)
    }

    val, err := rdb.Get(ctx, "key").Result()
    if err != nil {
        t.Fatal(err)
    }
    if val != "value" {
        t.Errorf("expected value, got %s", val)
    }
}
```

## CI Pipeline Integration

Add to your GitHub Actions workflow:

```yaml
name: Integration Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run tests
        run: pytest tests/integration/ -v
```

Testcontainers uses the Docker daemon available on the runner - no extra setup needed.

## Redis-Specific Testcontainers Module

The `testcontainers.redis` module provides a `RedisContainer` with convenience methods:

```python
from testcontainers.redis import RedisContainer

with RedisContainer("redis:7-alpine") as r:
    print(r.get_connection_url())
    # redis://localhost:32768
```

## Summary

Testcontainers eliminates the need for pre-configured test infrastructure by starting real Redis Docker containers programmatically within test code. This approach works identically in local development and CI, produces isolated test environments, and automatically cleans up containers after tests complete. It is available for Java, Python, Go, and other languages through official or community libraries.
