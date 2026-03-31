# How to Install and Set Up go-redis in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Go, go-redis

Description: Learn how to add go-redis to a Go project, configure a connection to Redis, and verify the setup with basic get and set operations.

---

go-redis is the most widely used Redis client for Go. It supports single-node, cluster, and Sentinel modes, and provides a type-safe API that maps closely to Redis commands.

## Install the Package

```bash
go get github.com/redis/go-redis/v9
```

## Create a Client

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()

    rdb := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "",  // leave empty if no auth
        DB:       0,   // default database
    })

    // Verify connection
    pong, err := rdb.Ping(ctx).Result()
    if err != nil {
        log.Fatalf("could not connect to Redis: %v", err)
    }
    fmt.Println(pong) // PONG
}
```

## Basic Get and Set

```go
// Set a key with no expiration
err = rdb.Set(ctx, "greeting", "hello", 0).Err()
if err != nil {
    log.Fatal(err)
}

// Get the value
val, err := rdb.Get(ctx, "greeting").Result()
if err == redis.Nil {
    fmt.Println("key does not exist")
} else if err != nil {
    log.Fatal(err)
} else {
    fmt.Println(val) // hello
}
```

## Set with TTL

```go
import "time"

err = rdb.Set(ctx, "token:abc", "user:42", 30*time.Minute).Err()
```

## Connection Options

```go
rdb := redis.NewClient(&redis.Options{
    Addr:         "localhost:6379",
    Password:     "mypassword",
    DB:           0,
    DialTimeout:  5 * time.Second,
    ReadTimeout:  3 * time.Second,
    WriteTimeout: 3 * time.Second,
    PoolSize:     10,
    MinIdleConns: 2,
})
```

## Using Environment Variables

```go
import "os"

rdb := redis.NewClient(&redis.Options{
    Addr:     os.Getenv("REDIS_ADDR"),     // e.g., "localhost:6379"
    Password: os.Getenv("REDIS_PASSWORD"),
})
```

## Parse a Redis URL

```go
opt, err := redis.ParseURL("redis://:password@localhost:6379/0")
if err != nil {
    log.Fatal(err)
}
rdb := redis.NewClient(opt)
```

## Handling redis.Nil

When a key does not exist, go-redis returns the sentinel error `redis.Nil`:

```go
val, err := rdb.Get(ctx, "missing:key").Result()
if err == redis.Nil {
    // Key does not exist - handle gracefully
    fmt.Println("cache miss")
} else if err != nil {
    log.Printf("redis error: %v", err)
}
```

Always check for `redis.Nil` before treating the error as a real failure.

## Summary

go-redis is installed with a single `go get` command and configured through `redis.Options`. The `Addr`, `Password`, and `DB` fields cover most setups, while `PoolSize`, timeout fields, and URL parsing handle production requirements. Distinguishing between `redis.Nil` (key missing) and real errors is a key pattern to get right from the start.
