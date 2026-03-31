# How to Mock Redis in Go Unit Tests

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Go, Testing

Description: Learn how to mock Redis in Go unit tests using go-redis/redismock to stub commands and verify interactions without a real Redis server.

---

Testing Redis-backed code without a real server keeps tests fast and deterministic. The `go-redis/redismock` package provides a mock that implements the same interface as `redis.Client`, letting you stub responses and verify that your code sends the right commands.

## Install

```bash
go get github.com/go-redis/redismock/v9
```

## Basic Mock Setup

```go
package main_test

import (
    "context"
    "testing"
    "time"

    "github.com/go-redis/redismock/v9"
    "github.com/redis/go-redis/v9"
)

func TestSetAndGet(t *testing.T) {
    ctx := context.Background()
    db, mock := redismock.NewClientMock()

    // Stub SET
    mock.ExpectSet("greeting", "hello", time.Hour).SetVal("OK")

    // Stub GET
    mock.ExpectGet("greeting").SetVal("hello")

    // Run commands
    db.Set(ctx, "greeting", "hello", time.Hour)
    val, err := db.Get(ctx, "greeting").Result()

    if err != nil {
        t.Fatal(err)
    }
    if val != "hello" {
        t.Errorf("expected hello, got %s", val)
    }

    // Verify all expectations were met
    if err := mock.ExpectationsWereMet(); err != nil {
        t.Error(err)
    }
}
```

## Testing a Service That Uses Redis

Service under test:

```go
type CacheService struct {
    rdb *redis.Client
}

func (s *CacheService) GetUser(ctx context.Context, id string) (string, error) {
    val, err := s.rdb.Get(ctx, "user:"+id).Result()
    if err == redis.Nil {
        return "", nil
    }
    return val, err
}

func (s *CacheService) SetUser(ctx context.Context, id, data string) error {
    return s.rdb.Set(ctx, "user:"+id, data, 30*time.Minute).Err()
}
```

Test:

```go
func TestCacheService_GetUser(t *testing.T) {
    ctx := context.Background()
    db, mock := redismock.NewClientMock()
    svc := &CacheService{rdb: db}

    // Test cache hit
    mock.ExpectGet("user:42").SetVal(`{"name":"Alice"}`)
    result, err := svc.GetUser(ctx, "42")
    if err != nil || result != `{"name":"Alice"}` {
        t.Errorf("unexpected result: %s %v", result, err)
    }

    // Test cache miss
    mock.ExpectGet("user:99").RedisNil()
    result, err = svc.GetUser(ctx, "99")
    if err != nil || result != "" {
        t.Errorf("expected empty result on cache miss, got: %s", result)
    }

    if err := mock.ExpectationsWereMet(); err != nil {
        t.Error(err)
    }
}
```

## Mocking Errors

```go
mock.ExpectGet("key").SetErr(fmt.Errorf("connection refused"))

_, err := db.Get(ctx, "key").Result()
if err == nil {
    t.Error("expected error but got none")
}
```

## Mocking TTL and Expiry

```go
mock.ExpectTTL("session:abc").SetVal(25 * time.Minute)
mock.ExpectExpire("session:abc", time.Hour).SetVal(true)
```

## Pipeline Mock

```go
pipe := db.Pipeline()
mock.ExpectSet("k1", "v1", 0).SetVal("OK")
mock.ExpectSet("k2", "v2", 0).SetVal("OK")

pipe.Set(ctx, "k1", "v1", 0)
pipe.Set(ctx, "k2", "v2", 0)
_, err := pipe.Exec(ctx)
```

## Summary

`go-redis/redismock` creates a test double for `redis.Client` that stubs command responses with `Expect*` methods. Calling `ExpectationsWereMet` at the end of each test confirms your code sent all expected commands. Use `RedisNil()` to simulate cache misses and `SetErr` to simulate connection failures - both are common paths that need test coverage.
