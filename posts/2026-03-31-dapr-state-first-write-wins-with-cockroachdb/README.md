# How to Use Dapr State First-Write-Wins with CockroachDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, CockroachDB, State Management, Concurrency, Distributed Systems

Description: Learn how to configure Dapr state management with first-write-wins concurrency using CockroachDB as the state store backend.

---

## What Is First-Write-Wins Concurrency

Dapr state management supports two concurrency models: last-write-wins (LWW) and first-write-wins (FWW). With FWW, once a key is written, subsequent writes fail if another write has already occurred concurrently. This is useful when only the first writer should succeed, such as initializing configuration or claiming a distributed lock.

CockroachDB is a distributed SQL database that works well with Dapr's FWW model because it provides strong serializable transactions with distributed consistency guarantees.

## Setting Up CockroachDB as a Dapr State Store

First, deploy CockroachDB and configure it as a Dapr component.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: cockroachdb-state
  namespace: default
spec:
  type: state.cockroachdb
  version: v1
  metadata:
  - name: connectionString
    value: "host=cockroachdb-public.default.svc.cluster.local user=root password='' dbname=daprstate port=26257 connect_timeout=10 sslmode=disable"
  - name: tableName
    value: state
```

Apply this component definition to your Kubernetes cluster:

```bash
kubectl apply -f cockroachdb-state.yaml
```

## Enabling First-Write-Wins in Your Application

When writing state with FWW concurrency, set the concurrency mode to `first-write` in your state options. Below is an example using the Dapr Go SDK:

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
    storeName := "cockroachdb-state"
    key := "config-initialized"
    value := []byte(`{"version": "1.0", "initialized": true}`)

    // Attempt first-write-wins: only the first caller succeeds
    err = client.SaveState(ctx, storeName, key, value, map[string]string{
        "concurrency": "first-write",
        "consistency": "strong",
    })
    if err != nil {
        fmt.Println("Write failed - key already exists or conflict:", err)
        return
    }
    fmt.Println("State written successfully (first write wins)")
}
```

## How Dapr Enforces First-Write-Wins

Dapr uses ETags internally to implement FWW. When you save state with `first-write` concurrency and no ETag, Dapr attempts an insert that fails if the key already exists. CockroachDB enforces this at the database level using its serializable isolation.

```sql
-- Dapr internally executes something similar to:
INSERT INTO state (key, value, etag, updatetime)
VALUES ($1, $2, $3, NOW())
ON CONFLICT (key) DO NOTHING;
-- Returns 0 rows affected if key exists = conflict
```

## Handling Concurrency Conflicts

Your application should handle the case where a write is rejected due to FWW conflict:

```go
func initializeConfig(ctx context.Context, client dapr.Client, storeName string) error {
    config := map[string]interface{}{
        "feature_flags": map[string]bool{
            "new_ui": false,
            "beta":   true,
        },
    }

    data, _ := json.Marshal(config)

    err := client.SaveState(ctx, storeName, "app-config", data, map[string]string{
        "concurrency": "first-write",
    })
    if err != nil {
        if strings.Contains(err.Error(), "conflict") ||
            strings.Contains(err.Error(), "etag mismatch") {
            // Another instance already initialized the config
            fmt.Println("Config already initialized by another instance, skipping")
            return nil
        }
        return fmt.Errorf("unexpected error saving config: %w", err)
    }

    fmt.Println("Config initialized successfully")
    return nil
}
```

## Practical Use Case: Distributed Leader Election

FWW with CockroachDB is ideal for simple leader election. Only one service instance wins the initial write:

```go
func tryBecomeLeader(ctx context.Context, client dapr.Client, instanceID string) (bool, error) {
    leaderData := map[string]string{
        "leader":    instanceID,
        "timestamp": time.Now().UTC().Format(time.RFC3339),
    }
    data, _ := json.Marshal(leaderData)

    err := client.SaveState(ctx, "cockroachdb-state", "leader", data, map[string]string{
        "concurrency": "first-write",
    })
    if err != nil {
        // Another instance became leader
        return false, nil
    }
    return true, nil
}

func main() {
    // Each instance tries to become leader at startup
    instanceID := os.Getenv("POD_NAME")
    isLeader, err := tryBecomeLeader(context.Background(), client, instanceID)
    if err != nil {
        log.Fatal(err)
    }
    if isLeader {
        fmt.Printf("Instance %s is now the leader\n", instanceID)
        runLeaderTasks()
    } else {
        fmt.Printf("Instance %s is a follower\n", instanceID)
        runFollowerTasks()
    }
}
```

## Verifying State in CockroachDB

You can inspect the written state directly in CockroachDB to confirm FWW behavior:

```sql
-- Connect to CockroachDB
cockroach sql --insecure --host=localhost:26257

-- View state table
SELECT key, etag, updatetime FROM daprstate.state WHERE key = 'leader';
```

## Testing FWW Behavior Locally

Run two instances simultaneously and observe that only one succeeds:

```bash
# Start two instances at the same time
dapr run --app-id instance-1 --dapr-http-port 3500 -- go run main.go &
dapr run --app-id instance-2 --dapr-http-port 3501 -- go run main.go &
wait

# Check which instance became leader
dapr invoke --app-id instance-1 --method leader --verb GET
```

## Summary

Dapr's first-write-wins concurrency model, backed by CockroachDB, provides a reliable mechanism for initializing shared state, claiming distributed locks, and performing leader election. The combination of Dapr's state API and CockroachDB's distributed SQL guarantees that exactly one writer succeeds in a race, making it a solid foundation for distributed coordination patterns.
