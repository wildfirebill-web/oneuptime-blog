# How to Implement Event Replay from Dapr Event Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Event Sourcing, Replay, State Management, Architecture

Description: Replay domain events from a Dapr-backed event store to rebuild projections, fix corrupted read models, or migrate aggregate state to a new schema.

---

Event replay is one of the most powerful features of event sourcing. When you need to rebuild a read model, fix a bug in a projection handler, or migrate to a new schema, you replay all historical events through updated handlers. Dapr's state management API provides the foundation for retrieving and replaying stored events.

## When to Replay Events

Common reasons to replay events:
- A projection handler had a bug and needs to be recomputed
- A new read model needs to be built from historical data
- An aggregate's state schema changed and needs migration
- Auditing or debugging requires reviewing past state changes

## Full Event Store Replay

Iterate over all events in the store and apply them to a new projection:

```go
// replay.go
package replay

import (
    "context"
    "encoding/json"
    "fmt"
    dapr "github.com/dapr/go-sdk/client"
)

type ReplayHandler func(event Event) error

func ReplayAll(client dapr.Client, aggregateType string, handler ReplayHandler) error {
    // In a real system, maintain an index of all aggregate IDs
    aggregateIDs, err := loadAggregateIDs(aggregateType)
    if err != nil {
        return fmt.Errorf("failed to load aggregate IDs: %w", err)
    }

    for _, id := range aggregateIDs {
        if err := ReplayAggregate(client, aggregateType, id, handler); err != nil {
            return fmt.Errorf("replay failed for %s/%s: %w", aggregateType, id, err)
        }
    }
    return nil
}

func ReplayAggregate(client dapr.Client, aggregateType, aggregateID string, handler ReplayHandler) error {
    for seq := int64(1); ; seq++ {
        key := fmt.Sprintf("%s:%s:%d", aggregateType, aggregateID, seq)
        item, err := client.GetState(context.Background(), "event-store", key, nil)
        if err != nil || item == nil || len(item.Value) == 0 {
            break
        }

        var event Event
        if err := json.Unmarshal(item.Value, &event); err != nil {
            return fmt.Errorf("failed to deserialize event %s: %w", key, err)
        }

        if err := handler(event); err != nil {
            return fmt.Errorf("handler failed for event %s: %w", key, err)
        }
    }
    return nil
}
```

## Rebuilding a Read Model

```go
// rebuild_read_model.go

type OrderSummary struct {
    OrderID    string
    CustomerID string
    Status     string
    Total      float64
}

func RebuildOrderSummaries(client dapr.Client) error {
    summaries := map[string]*OrderSummary{}

    return ReplayAll(client, "Order", func(event Event) error {
        switch event.EventType {
        case "OrderCreated":
            summaries[event.AggregateID] = &OrderSummary{
                OrderID:    event.AggregateID,
                CustomerID: event.Payload["customerId"].(string),
                Status:     "pending",
                Total:      event.Payload["total"].(float64),
            }
        case "OrderPaid":
            if s, ok := summaries[event.AggregateID]; ok {
                s.Status = "paid"
            }
        case "OrderShipped":
            if s, ok := summaries[event.AggregateID]; ok {
                s.Status = "shipped"
            }
        }
        return nil
    })
}
```

## Partial Replay from a Timestamp

Replay only events that occurred after a specific time to update a projection incrementally:

```go
func ReplayFrom(client dapr.Client, aggregateType, aggregateID string, afterSeq int64, handler ReplayHandler) error {
    for seq := afterSeq + 1; ; seq++ {
        key := fmt.Sprintf("%s:%s:%d", aggregateType, aggregateID, seq)
        item, _ := client.GetState(context.Background(), "event-store", key, nil)
        if item == nil || len(item.Value) == 0 {
            break
        }
        var event Event
        json.Unmarshal(item.Value, &event)
        handler(event)
    }
    return nil
}
```

## Tracking Replay Progress

For large event stores, track replay progress so you can resume after failure:

```go
func SaveReplayCheckpoint(client dapr.Client, projectionName string, lastSeq int64) error {
    return client.SaveState(context.Background(), "snapshot-store",
        fmt.Sprintf("replay-checkpoint:%s", projectionName),
        map[string]int64{"lastSeq": lastSeq}, nil)
}
```

## Summary

Event replay from a Dapr event store enables rebuilding projections, fixing corrupted read models, and migrating aggregate schemas. By iterating over stored events and applying them through updated handlers, you leverage the immutability of your event log to correct mistakes and evolve your system without data loss.
