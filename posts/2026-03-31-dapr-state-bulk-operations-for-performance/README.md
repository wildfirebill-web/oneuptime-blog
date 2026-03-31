# How to Use Dapr State Bulk Operations for Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, State Management, Performance, Bulk Operations, Distributed Systems

Description: Learn how to use Dapr bulk state operations to dramatically reduce latency and improve throughput when reading and writing multiple state keys.

---

## Why Bulk Operations Matter

When your application needs to read or write multiple state keys, making individual API calls for each key adds significant network overhead. Dapr's bulk state operations let you batch multiple reads and writes into a single API call, reducing round-trips and improving overall throughput.

A typical scenario: loading a user's profile requires reading 10+ keys (preferences, settings, metadata). With single operations, that is 10 API calls. With bulk operations, it is 1.

## Bulk Get: Reading Multiple Keys at Once

The bulk get operation retrieves multiple state keys in a single request.

```go
package main

import (
    "context"
    "fmt"
    "log"

    dapr "github.com/dapr/go-sdk/client"
)

func main() {
    client, err := dapr.NewClient()
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    ctx := context.Background()
    storeName := "statestore"

    // Fetch multiple keys at once
    keys := []string{
        "user:1001:profile",
        "user:1001:settings",
        "user:1001:preferences",
        "user:1001:notifications",
    }

    items, err := client.GetBulkState(ctx, storeName, keys, nil, 10)
    if err != nil {
        log.Fatal(err)
    }

    for _, item := range items {
        if item.Error != "" {
            fmt.Printf("Error fetching key %s: %s\n", item.Key, item.Error)
            continue
        }
        fmt.Printf("Key: %s, Value: %s\n", item.Key, string(item.Value))
    }
}
```

## Bulk Save: Writing Multiple Keys at Once

The bulk save operation writes multiple keys in a single API call.

```go
func saveUserData(ctx context.Context, client dapr.Client, userID string) error {
    storeName := "statestore"

    items := []*dapr.SetStateItem{
        {
            Key:   fmt.Sprintf("user:%s:profile", userID),
            Value: []byte(`{"name":"Alice","email":"alice@example.com"}`),
        },
        {
            Key:   fmt.Sprintf("user:%s:settings", userID),
            Value: []byte(`{"theme":"dark","language":"en"}`),
        },
        {
            Key:   fmt.Sprintf("user:%s:preferences", userID),
            Value: []byte(`{"notifications":true,"marketing":false}`),
        },
    }

    err := client.SaveBulkState(ctx, storeName, items...)
    if err != nil {
        return fmt.Errorf("bulk save failed: %w", err)
    }

    fmt.Printf("Saved %d items for user %s\n", len(items), userID)
    return nil
}
```

## Bulk Delete: Removing Multiple Keys

```go
func deleteUserData(ctx context.Context, client dapr.Client, userID string) error {
    storeName := "statestore"

    keys := []dapr.DeleteStateItem{
        {Key: fmt.Sprintf("user:%s:profile", userID)},
        {Key: fmt.Sprintf("user:%s:settings", userID)},
        {Key: fmt.Sprintf("user:%s:preferences", userID)},
    }

    err := client.DeleteBulkState(ctx, storeName, keys)
    if err != nil {
        return fmt.Errorf("bulk delete failed: %w", err)
    }

    fmt.Printf("Deleted user data for %s\n", userID)
    return nil
}
```

## Using the HTTP API for Bulk Operations

If you are not using a Dapr SDK, the HTTP API supports bulk operations directly:

```bash
# Bulk get via HTTP
curl -X POST http://localhost:3500/v1.0/state/statestore/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "keys": ["user:1001:profile", "user:1001:settings"],
    "parallelism": 5
  }'
```

```bash
# Bulk save via HTTP
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[
    {"key": "user:1001:profile", "value": {"name": "Alice"}},
    {"key": "user:1001:settings", "value": {"theme": "dark"}}
  ]'
```

## Controlling Parallelism

The `parallelism` parameter on bulk get controls how many keys are fetched concurrently. Tune this based on your state store capacity:

```go
// Fetch 100 keys with parallelism of 20
items, err := client.GetBulkState(ctx, storeName, keys, nil, 20)
```

For Redis, a parallelism of 10-20 is typically safe. For CockroachDB or PostgreSQL, keep it lower (5-10) to avoid connection saturation.

## Benchmarking Single vs. Bulk Operations

Here is a simple benchmark showing the performance difference:

```go
func benchmarkSingleVsBulk(ctx context.Context, client dapr.Client) {
    keys := make([]string, 50)
    for i := range keys {
        keys[i] = fmt.Sprintf("bench-key-%d", i)
    }

    // Single operations
    start := time.Now()
    for _, key := range keys {
        client.GetState(ctx, "statestore", key, nil)
    }
    singleDuration := time.Since(start)

    // Bulk operation
    start = time.Now()
    client.GetBulkState(ctx, "statestore", keys, nil, 10)
    bulkDuration := time.Since(start)

    fmt.Printf("Single ops: %v\n", singleDuration)
    fmt.Printf("Bulk ops:   %v\n", bulkDuration)
    fmt.Printf("Speedup:    %.2fx\n", float64(singleDuration)/float64(bulkDuration))
}
```

## Combining Bulk Operations with Transactions

For atomic writes across multiple keys, combine bulk save with Dapr's state transactions:

```go
func transactionalBulkSave(ctx context.Context, client dapr.Client) error {
    ops := []*dapr.StateOperation{
        {
            Type: dapr.StateOperationTypeUpsert,
            Item: &dapr.SetStateItem{
                Key:   "order:1001",
                Value: []byte(`{"status":"confirmed","total":99.99}`),
            },
        },
        {
            Type: dapr.StateOperationTypeUpsert,
            Item: &dapr.SetStateItem{
                Key:   "inventory:SKU-42",
                Value: []byte(`{"quantity":49}`),
            },
        },
        {
            Type: dapr.StateOperationTypeDelete,
            Item: &dapr.SetStateItem{
                Key: "cart:user:1001",
            },
        },
    }

    return client.ExecuteStateTransaction(ctx, "statestore", nil, ops)
}
```

## Summary

Dapr bulk state operations are one of the most effective ways to improve the performance of stateful microservices. By batching reads and writes into single API calls, you reduce network round-trips and leverage the underlying state store's batch capabilities. Use bulk get for loading multiple related keys, bulk save for persisting related data together, and combine with transactions when atomicity is required.
