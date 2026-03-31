# How to Implement Redlock in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redlock, Go, Distributed Locks, Concurrency, Golang

Description: Implement Redlock distributed locking in Go using the redsync library across multiple Redis instances for reliable fault-tolerant mutual exclusion in distributed systems.

---

## Install Dependencies

```bash
go get github.com/go-redsync/redsync/v4
go get github.com/go-redsync/redsync/v4/redis/goredis/v9
go get github.com/redis/go-redis/v9
```

## Single-Node Redlock

```go
package main

import (
    "context"
    "fmt"
    "time"

    goredislib "github.com/redis/go-redis/v9"
    "github.com/go-redsync/redsync/v4"
    "github.com/go-redsync/redsync/v4/redis/goredis/v9"
)

func main() {
    client := goredislib.NewClient(&goredislib.Options{
        Addr: "localhost:6379",
    })

    pool := goredis.NewPool(client)
    rs := redsync.New(pool)

    mutex := rs.NewMutex("lock:resource:order-5001",
        redsync.WithExpiry(10*time.Second),
        redsync.WithTries(10),
        redsync.WithRetryDelay(200*time.Millisecond),
    )

    ctx := context.Background()

    if err := mutex.LockContext(ctx); err != nil {
        fmt.Printf("Failed to acquire lock: %v\n", err)
        return
    }

    defer func() {
        if ok, err := mutex.UnlockContext(ctx); !ok || err != nil {
            fmt.Printf("Failed to release lock: %v\n", err)
        }
    }()

    fmt.Println("Lock acquired - processing...")
    time.Sleep(2 * time.Second)
    fmt.Println("Processing complete")
}
```

## Multi-Node Redlock

```go
package main

import (
    "context"
    "fmt"
    "time"

    goredislib "github.com/redis/go-redis/v9"
    "github.com/go-redsync/redsync/v4"
    "github.com/go-redsync/redsync/v4/redis/goredis/v9"
)

func setupRedsync() *redsync.Redsync {
    addrs := []string{
        "redis-1:6379",
        "redis-2:6379",
        "redis-3:6379",
        "redis-4:6379",
        "redis-5:6379",
    }

    pools := make([]redsync.Pool, len(addrs))
    for i, addr := range addrs {
        client := goredislib.NewClient(&goredislib.Options{Addr: addr})
        pools[i] = goredis.NewPool(client)
    }

    return redsync.New(pools...)
}

func processOrder(rs *redsync.Redsync, orderID string) error {
    mutex := rs.NewMutex(
        fmt.Sprintf("lock:order:%s", orderID),
        redsync.WithExpiry(15*time.Second),
        redsync.WithTries(5),
        redsync.WithRetryDelay(300*time.Millisecond),
        redsync.WithDriftFactor(0.01),
    )

    ctx := context.Background()
    if err := mutex.LockContext(ctx); err != nil {
        return fmt.Errorf("failed to acquire lock: %w", err)
    }
    defer mutex.UnlockContext(ctx)

    fmt.Printf("Processing order %s...\n", orderID)
    time.Sleep(500 * time.Millisecond)
    fmt.Printf("Order %s processed\n", orderID)
    return nil
}

func main() {
    rs := setupRedsync()
    if err := processOrder(rs, "5001"); err != nil {
        fmt.Println("Error:", err)
    }
}
```

## Extending a Lock

```go
func longRunningTask(rs *redsync.Redsync, resource string) error {
    mutex := rs.NewMutex(resource, redsync.WithExpiry(10*time.Second))

    ctx := context.Background()
    if err := mutex.LockContext(ctx); err != nil {
        return err
    }
    defer mutex.UnlockContext(ctx)

    for i := 0; i < 5; i++ {
        time.Sleep(2 * time.Second)
        if ok, err := mutex.Extend(); !ok || err != nil {
            return fmt.Errorf("failed to extend lock: %w", err)
        }
        fmt.Printf("Step %d complete, lock extended\n", i+1)
    }
    return nil
}
```

## Concurrent Lock Contention Test

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"

    goredislib "github.com/redis/go-redis/v9"
    "github.com/go-redsync/redsync/v4"
    "github.com/go-redsync/redsync/v4/redis/goredis/v9"
)

func testConcurrentAccess() {
    client := goredislib.NewClient(&goredislib.Options{Addr: "localhost:6379"})
    pool := goredis.NewPool(client)
    rs := redsync.New(pool)

    var wg sync.WaitGroup
    successCount := 0
    failCount := 0
    var mu sync.Mutex

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            mutex := rs.NewMutex("lock:shared:resource",
                redsync.WithExpiry(3*time.Second),
                redsync.WithTries(3),
                redsync.WithRetryDelay(100*time.Millisecond),
            )

            if err := mutex.LockContext(context.Background()); err != nil {
                mu.Lock()
                failCount++
                mu.Unlock()
                return
            }

            defer mutex.UnlockContext(context.Background())
            time.Sleep(200 * time.Millisecond)

            mu.Lock()
            successCount++
            mu.Unlock()
            fmt.Printf("Worker %d completed critical section\n", id)
        }(i)
    }

    wg.Wait()
    fmt.Printf("Success: %d, Failed to acquire: %d\n", successCount, failCount)
}
```

## Error Handling Best Practices

```go
func safeProcess(rs *redsync.Redsync, resource string, work func() error) error {
    mutex := rs.NewMutex(resource,
        redsync.WithExpiry(10*time.Second),
        redsync.WithTries(5),
    )

    if err := mutex.Lock(); err != nil {
        return fmt.Errorf("lock acquisition failed: %w", err)
    }

    workErr := work()

    if ok, unlockErr := mutex.Unlock(); !ok || unlockErr != nil {
        // Log unlock failure but don't override work error
        fmt.Printf("Warning: failed to release lock: %v\n", unlockErr)
    }

    return workErr
}
```

## Summary

Go implements Redlock using the `redsync` library, which is backed by `go-redis` clients. The `NewMutex` function creates a mutex with configurable expiry, retry count, and retry delay. For production fault tolerance, pass 5 independent Redis clients to `redsync.New`. The `Extend` method prolongs lock validity for long-running tasks, and `UnlockContext` uses a Lua script for atomic check-and-delete to prevent releasing a lock owned by another process.
