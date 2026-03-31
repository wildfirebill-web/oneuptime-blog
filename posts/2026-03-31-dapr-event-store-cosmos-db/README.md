# How to Implement Event Store with Dapr and Cosmos DB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Event Sourcing, Cosmos DB, Azure, State Management

Description: Build a globally distributed event store using Dapr state management backed by Azure Cosmos DB with partition keys optimized for event stream queries.

---

## Overview

Azure Cosmos DB offers single-digit millisecond latency, global distribution, and strong consistency guarantees - all valuable properties for an event store. Dapr's Cosmos DB state store component maps state keys directly to Cosmos DB documents, making it straightforward to implement an ordered, append-only event log.

## Configure the Cosmos DB State Store

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: event-store
spec:
  type: state.azure.cosmosdb
  version: v1
  metadata:
    - name: url
      value: "https://myaccount.documents.azure.com:443/"
    - name: masterKey
      secretKeyRef:
        name: cosmos-secret
        key: masterKey
    - name: database
      value: "EventStore"
    - name: collection
      value: "DomainEvents"
    - name: consistencyLevel
      value: "Strong"
    - name: contentType
      value: "application/json"
```

Use `consistencyLevel: Strong` to ensure all reads reflect the latest committed writes, which is critical for detecting concurrency conflicts.

## Event Schema

Define a structured event document:

```go
// event.go
package eventstore

import "time"

type DomainEvent struct {
    AggregateType string                 `json:"aggregateType"`
    AggregateID   string                 `json:"aggregateId"`
    Sequence      int64                  `json:"sequence"`
    EventType     string                 `json:"eventType"`
    Payload       map[string]interface{} `json:"payload"`
    CorrelationID string                 `json:"correlationId"`
    OccurredAt    time.Time              `json:"occurredAt"`
}
```

## Appending Events

Use Dapr's ETag-based concurrency to ensure only the first writer succeeds for a given sequence number:

```go
// appender.go
package eventstore

import (
    "context"
    "fmt"
    dapr "github.com/dapr/go-sdk/client"
)

func AppendEvent(ctx context.Context, client dapr.Client, event DomainEvent) error {
    // Key format: aggregateType||aggregateId||paddedSequence
    key := fmt.Sprintf("%s||%s||%010d",
        event.AggregateType,
        event.AggregateID,
        event.Sequence,
    )

    opts := &dapr.StateOptions{
        Concurrency: dapr.StateConcurrencyFirstWrite,
        Consistency: dapr.StateConsistencyStrong,
    }

    return client.SaveStateWithETag(ctx, "event-store", key, event, "", nil, opts)
}
```

## Reading Event Streams

Iterate over sequence numbers to reconstruct the full event stream:

```go
func ReadStream(ctx context.Context, client dapr.Client, aggregateType, aggregateID string, fromSeq int64) ([]DomainEvent, error) {
    var events []DomainEvent

    for seq := fromSeq; ; seq++ {
        key := fmt.Sprintf("%s||%s||%010d", aggregateType, aggregateID, seq)
        item, err := client.GetState(ctx, "event-store", key, map[string]string{
            "consistency": "strong",
        })
        if err != nil || item == nil || len(item.Value) == 0 {
            break
        }

        var evt DomainEvent
        if jsonErr := json.Unmarshal(item.Value, &evt); jsonErr != nil {
            return nil, jsonErr
        }
        events = append(events, evt)
    }

    return events, nil
}
```

## Transactional Multi-Event Append

When a command produces multiple events, use Dapr's transactional state API:

```go
func AppendEvents(ctx context.Context, client dapr.Client, events []DomainEvent) error {
    ops := make([]*dapr.StateOperation, 0, len(events))
    for _, event := range events {
        key := fmt.Sprintf("%s||%s||%010d",
            event.AggregateType, event.AggregateID, event.Sequence)
        ops = append(ops, &dapr.StateOperation{
            Type: dapr.StateOperationTypeUpsert,
            Item: &dapr.SetStateItem{
                Key:   key,
                Value: event,
                Options: &dapr.StateOptions{
                    Concurrency: dapr.StateConcurrencyFirstWrite,
                },
            },
        })
    }

    return client.ExecuteStateTransaction(ctx, "event-store", nil, ops)
}
```

## Global Distribution

To replicate event streams across Azure regions, configure multi-region writes in Cosmos DB and point each region's Dapr sidecar to the local endpoint:

```yaml
metadata:
  - name: url
    value: "https://myaccount-eastus.documents.azure.com:443/"
  - name: preferredLocations
    value: "East US,West Europe,Southeast Asia"
```

## Summary

Azure Cosmos DB's strong consistency, global distribution, and partition-based scaling make it an excellent backend for a Dapr-powered event store. Sequential key design prevents duplicate events, transactional state operations enable multi-event appends atomically, and the Cosmos DB multi-region write capability keeps event streams replicated worldwide with low latency.
