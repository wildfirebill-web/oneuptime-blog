# How to Use Redis Pipelining in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Go, Pipelining

Description: Learn how to use Redis pipelining in Go with go-redis to batch multiple commands in a single network round-trip for higher throughput.

---

Redis pipelining sends multiple commands to the server without waiting for individual responses. This reduces round-trip latency when you need to execute many commands together. go-redis supports pipelining through `Pipelined` and `TxPipelined`.

## Basic Pipeline

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    defer rdb.Close()

    pipe := rdb.Pipeline()

    setCmd := pipe.Set(ctx, "key1", "value1", time.Hour)
    getCmd := pipe.Get(ctx, "key1")
    incrCmd := pipe.Incr(ctx, "counter")

    _, err := pipe.Exec(ctx)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Println(setCmd.Val())  // OK
    fmt.Println(getCmd.Val())  // value1
    fmt.Println(incrCmd.Val()) // 1
}
```

## Using Pipelined (Callback Style)

```go
cmds, err := rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
    for i := 0; i < 100; i++ {
        key := fmt.Sprintf("item:%d", i)
        pipe.Set(ctx, key, fmt.Sprintf("val-%d", i), time.Hour)
    }
    return nil
})
if err != nil {
    log.Fatal(err)
}

fmt.Printf("Executed %d commands\n", len(cmds))
```

## Bulk Fetch with Pipeline

```go
keys := []string{"user:1", "user:2", "user:3", "user:4"}

var cmds []*redis.StringCmd

pipe := rdb.Pipeline()
for _, k := range keys {
    cmds = append(cmds, pipe.Get(ctx, k))
}

_, err := pipe.Exec(ctx)
// redis.Nil errors are expected for missing keys, not fatal
if err != nil && err != redis.Nil {
    log.Fatal(err)
}

for i, cmd := range cmds {
    val, err := cmd.Result()
    if err == redis.Nil {
        fmt.Printf("%s: (not found)\n", keys[i])
    } else {
        fmt.Printf("%s: %s\n", keys[i], val)
    }
}
```

## Transactional Pipeline (MULTI/EXEC)

Use `TxPipelined` when you need all commands to execute atomically:

```go
_, err = rdb.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
    pipe.Set(ctx, "balance:alice", "900", 0)
    pipe.Set(ctx, "balance:bob", "1100", 0)
    return nil
})
if err != nil {
    log.Fatal(err)
}
```

## Performance Comparison

Without pipelining, 1000 SET commands each incur a network round-trip. With pipelining:

```go
// Batch 1000 writes in one pipeline
_, err = rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
    for i := 0; i < 1000; i++ {
        pipe.Set(ctx, fmt.Sprintf("bulk:%d", i), i, 0)
    }
    return nil
})
```

This typically runs 10-50x faster than sending commands individually, depending on network latency.

## Pipeline vs Transaction

- `Pipeline` / `Pipelined` - batches commands, server may interleave other client commands between them. No atomicity guarantee.
- `TxPipelined` - wraps commands in `MULTI`/`EXEC`, all-or-nothing execution, but does not support `WATCH`.

Use `TxPipelined` only when atomicity matters; plain pipelining is sufficient for bulk reads and writes.

## Summary

go-redis pipelining batches multiple commands into a single network round-trip. Use the `Pipelined` callback for a clean API, or `Pipeline` directly when you need to reference command objects before execution. `TxPipelined` adds `MULTI`/`EXEC` wrapping for atomic execution. Pipelining is most beneficial when running many small commands where network latency dominates execution time.
