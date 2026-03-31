# How to Implement Event Store Compaction with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Event Sourcing, Compaction, State Management, Architecture

Description: Implement event store compaction strategies using Dapr state management to manage storage growth, archive old events, and keep active event streams performant.

---

Over time, an event store accumulates millions of events. Without a compaction strategy, storage costs grow unbounded and loading older aggregates becomes slow. Compaction removes, archives, or consolidates older events while preserving system correctness.

## Compaction Strategies

Three approaches for managing event store growth:

1. **Snapshot + truncation**: Take a snapshot and delete events before it
2. **Archival**: Move old events to cold storage while keeping recent events hot
3. **Retention policies**: Automatically expire events older than N days

## Strategy 1 - Snapshot and Truncate

After taking a snapshot, delete events up to the snapshot sequence:

```go
// compaction.go
package compaction

import (
    "context"
    "fmt"
    dapr "github.com/dapr/go-sdk/client"
)

func CompactEventStream(client dapr.Client, aggregateType, aggregateID string, upToSeq int64) error {
    // First, ensure a snapshot exists at upToSeq
    if err := verifySnapshot(client, aggregateType, aggregateID, upToSeq); err != nil {
        return fmt.Errorf("cannot compact without valid snapshot: %w", err)
    }

    // Delete events from sequence 1 to upToSeq
    for seq := int64(1); seq <= upToSeq; seq++ {
        key := fmt.Sprintf("%s:%s:%d", aggregateType, aggregateID, seq)
        if err := client.DeleteState(context.Background(), "event-store", key, nil); err != nil {
            return fmt.Errorf("failed to delete event %s: %w", key, err)
        }
    }

    return nil
}
```

## Strategy 2 - Archive to Cold Storage

Move old events to a low-cost storage tier before deleting them:

```go
func ArchiveAndCompact(client dapr.Client, archiveClient dapr.Client,
    aggregateType, aggregateID string, beforeSeq int64) error {

    for seq := int64(1); seq < beforeSeq; seq++ {
        key := fmt.Sprintf("%s:%s:%d", aggregateType, aggregateID, seq)

        // Read the event
        item, err := client.GetState(context.Background(), "event-store", key, nil)
        if err != nil || len(item.Value) == 0 {
            break
        }

        // Write to archive store (e.g., a slower/cheaper backend)
        archiveKey := fmt.Sprintf("archive:%s", key)
        archiveClient.SaveState(context.Background(), "archive-store", archiveKey, item.Value, nil)

        // Delete from hot store
        client.DeleteState(context.Background(), "event-store", key, nil)
    }

    return nil
}
```

## Strategy 3 - TTL-Based Expiration

Configure TTL on events when they are stored. Dapr state stores that support TTL will automatically expire old events:

```go
func AppendEventWithTTL(client dapr.Client, event Event, ttlSeconds int) error {
    key := fmt.Sprintf("%s:%s:%d", event.AggregateType, event.AggregateID, event.Sequence)

    return client.SaveStateWithETag(
        context.Background(),
        "event-store",
        key,
        event,
        "",
        map[string]string{
            "ttlInSeconds": fmt.Sprintf("%d", ttlSeconds),
        },
        nil,
    )
}
```

Note: Only use TTL for events where you have taken a snapshot. Expiring events without snapshots results in unrecoverable data loss.

## Running Compaction as a Scheduled Job

Run compaction as a Dapr jobs API scheduled task:

```yaml
# Schedule compaction every night at 2 AM
apiVersion: dapr.io/v1alpha1
kind: Job
metadata:
  name: event-compaction
spec:
  schedule: "0 2 * * *"
  data:
    aggregateTypes: ["Order", "User", "Product"]
    retentionDays: 90
```

```go
// Job handler
http.HandleFunc("/jobs/event-compaction", func(w http.ResponseWriter, r *http.Request) {
    var config CompactionConfig
    json.NewDecoder(r.Body).Decode(&config)

    cutoff := time.Now().AddDate(0, 0, -config.RetentionDays)
    for _, aggType := range config.AggregateTypes {
        runCompactionForType(aggType, cutoff)
    }
    w.WriteHeader(http.StatusOK)
})
```

## Summary

Event store compaction with Dapr requires choosing between snapshot-and-truncate, archive-to-cold-storage, or TTL-based expiration depending on your retention and recovery requirements. Always verify that snapshots exist before deleting events, and run compaction as a scheduled background job to avoid impacting application performance.
