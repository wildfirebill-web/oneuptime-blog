# How to Use the Dapr Distributed Lock API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Distributed Lock, Concurrency, Microservice, API

Description: A practical introduction to the Dapr Distributed Lock API - learn how to acquire and release locks across microservices using the HTTP API and SDKs.

---

The Dapr Distributed Lock API provides a portable locking mechanism across microservices. It abstracts the underlying lock store (like Redis), giving services a consistent way to coordinate access to shared resources without tight coupling to a specific technology.

## When to Use Distributed Locks

Use the Dapr Distributed Lock API when you need to:
- Prevent duplicate processing of the same message or event
- Coordinate access to a shared external resource
- Implement leader election among service instances
- Guard critical sections across different microservices

## Component Setup

Define a lock store component using Redis:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: lockstore
spec:
  type: lock.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis:6379
  - name: redisPassword
    value: ""
```

## Acquiring a Lock via HTTP API

Send a POST request to acquire a lock:

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/lock/lockstore \
  -H "Content-Type: application/json" \
  -d '{
    "resourceId": "my-shared-resource",
    "lockOwner": "instance-abc123",
    "expiryInSeconds": 30
  }'
```

A successful response:

```json
{ "success": true }
```

If the lock is already held:

```json
{ "success": false }
```

## Releasing a Lock via HTTP API

Release the lock when the critical section is complete:

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/unlock/lockstore \
  -H "Content-Type: application/json" \
  -d '{
    "resourceId": "my-shared-resource",
    "lockOwner": "instance-abc123"
  }'
```

## Using the Go SDK

The Go SDK simplifies lock management:

```go
import (
    dapr "github.com/dapr/go-sdk/client"
    "context"
    "fmt"
)

func processWithLock(client dapr.Client) error {
    resp, err := client.TryLockAlpha1(context.Background(), "lockstore", &dapr.LockRequest{
        LockOwner:         "instance-abc123",
        ResourceID:        "my-shared-resource",
        ExpiryInSeconds:   30,
    })
    if err != nil {
        return err
    }
    if !resp.Success {
        fmt.Println("Lock not acquired - another instance is processing")
        return nil
    }
    defer client.UnlockAlpha1(context.Background(), "lockstore", &dapr.UnlockRequest{
        LockOwner:  "instance-abc123",
        ResourceID: "my-shared-resource",
    })
    // Critical section
    fmt.Println("Lock acquired - processing resource")
    return nil
}
```

## Key Parameters

| Parameter | Description |
|---|---|
| `resourceId` | Unique identifier for the resource being locked |
| `lockOwner` | Unique ID for the lock holder (use pod/instance ID) |
| `expiryInSeconds` | Automatic lock expiry to prevent deadlocks |

## Best Practices

Always set a meaningful `expiryInSeconds` to avoid deadlocks if the owner crashes. Generate a unique `lockOwner` per instance - using `os.Hostname()` or a UUID works well. Release locks in a `defer` or `finally` block to ensure cleanup even on errors.

## Summary

The Dapr Distributed Lock API provides a language-agnostic, store-independent way to implement mutual exclusion across microservices. With support for both HTTP and SDK interfaces, it integrates cleanly into any Dapr application. Always pair lock acquisition with explicit release and a reasonable expiry time to keep systems healthy under failure conditions.
