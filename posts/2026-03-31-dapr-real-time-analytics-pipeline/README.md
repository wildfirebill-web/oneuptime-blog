# How to Build a Real-Time Analytics Pipeline with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Analytics, Pub/Sub, Streaming, Pipeline

Description: Learn how to build a real-time analytics pipeline using Dapr pub/sub for event ingestion, state for aggregation, and bindings for data export.

---

## Overview

A real-time analytics pipeline ingests events, aggregates them on the fly, and exposes metrics dashboards. Dapr pub/sub handles high-throughput event ingestion, state stores maintain rolling aggregates, and output bindings export data to analytics backends.

## Pipeline Architecture

```
Event Sources --> Dapr Pub/Sub --> Aggregator Services --> Dashboard API
                                        |
                                   Dapr State (rolling windows)
                                        |
                                   Output Bindings --> ClickHouse / Elasticsearch
```

## Event Schema

```go
package main

type AnalyticsEvent struct {
    EventID   string            `json:"eventId"`
    EventType string            `json:"eventType"`
    UserID    string            `json:"userId"`
    SessionID string            `json:"sessionId"`
    Timestamp int64             `json:"timestamp"`
    Properties map[string]any  `json:"properties"`
}

type PageView struct {
    URL      string `json:"url"`
    Referrer string `json:"referrer"`
    Duration int    `json:"durationMs"`
}
```

## High-Throughput Event Ingestion

```go
func handleIngestEvent(w http.ResponseWriter, r *http.Request) {
    var event AnalyticsEvent
    json.NewDecoder(r.Body).Decode(&event)
    event.EventID = uuid.New().String()
    event.Timestamp = time.Now().UnixMilli()

    // Publish to analytics pipeline
    daprClient.PublishEvent(
        context.Background(),
        "analytics-pubsub",
        "raw-events",
        event,
    )

    w.WriteHeader(http.StatusAccepted)
}
```

## Rolling Window Aggregator

```go
type WindowAggregator struct {
    daprClient dapr.Client
    windowSize time.Duration
}

type WindowState struct {
    EventCount  int              `json:"eventCount"`
    UniqueUsers map[string]bool  `json:"uniqueUsers"`
    EventTypes  map[string]int   `json:"eventTypes"`
    WindowStart int64            `json:"windowStart"`
}

func (wa *WindowAggregator) HandleEvent(ctx context.Context, e *common.TopicEvent) (bool, error) {
    var event AnalyticsEvent
    json.Unmarshal(e.RawData, &event)

    // Determine current window bucket
    now := time.Now()
    windowKey := fmt.Sprintf("window:%s", now.Truncate(wa.windowSize).Format(time.RFC3339))

    // Load current window state
    item, _ := wa.daprClient.GetState(ctx, "analytics-store", windowKey, nil)
    var state WindowState
    if item.Value != nil {
        json.Unmarshal(item.Value, &state)
    } else {
        state = WindowState{
            UniqueUsers: make(map[string]bool),
            EventTypes:  make(map[string]int),
            WindowStart: now.Truncate(wa.windowSize).Unix(),
        }
    }

    // Update aggregates
    state.EventCount++
    state.UniqueUsers[event.UserID] = true
    state.EventTypes[event.EventType]++

    data, _ := json.Marshal(state)
    return false, wa.daprClient.SaveState(ctx, "analytics-store", windowKey, data, nil)
}
```

## Real-Time Metrics API

```go
func handleGetMetrics(w http.ResponseWriter, r *http.Request) {
    duration := r.URL.Query().Get("duration") // "1m", "5m", "1h"
    dur, _ := time.ParseDuration(duration)

    ctx := context.Background()
    now := time.Now()
    windowStart := now.Add(-dur)

    var totalEvents int
    uniqueUsers := map[string]bool{}
    eventTypes := map[string]int{}

    // Aggregate across time windows
    for t := windowStart; t.Before(now); t = t.Add(time.Minute) {
        windowKey := fmt.Sprintf("window:%s", t.Truncate(time.Minute).Format(time.RFC3339))
        item, _ := daprClient.GetState(ctx, "analytics-store", windowKey, nil)
        if item.Value == nil {
            continue
        }

        var state WindowState
        json.Unmarshal(item.Value, &state)
        totalEvents += state.EventCount
        for u := range state.UniqueUsers {
            uniqueUsers[u] = true
        }
        for et, count := range state.EventTypes {
            eventTypes[et] += count
        }
    }

    json.NewEncoder(w).Encode(map[string]any{
        "duration":    duration,
        "totalEvents": totalEvents,
        "uniqueUsers": len(uniqueUsers),
        "eventTypes":  eventTypes,
    })
}
```

## Exporting to ClickHouse

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: clickhouse-output
spec:
  type: bindings.http
  version: v1
  metadata:
    - name: url
      value: "http://clickhouse:8123"
```

```go
func exportToClickHouse(client dapr.Client, events []AnalyticsEvent) error {
    rows := buildInsertSQL(events)
    _, err := client.InvokeBinding(context.Background(), &dapr.InvokeBindingRequest{
        Name:      "clickhouse-output",
        Operation: "post",
        Data:      []byte(rows),
        Metadata: map[string]string{
            "path": "/?query=INSERT+INTO+analytics.events+FORMAT+JSONEachRow",
        },
    })
    return err
}
```

## Summary

Dapr enables a real-time analytics pipeline through pub/sub for high-throughput event ingestion, state stores for rolling window aggregations, and output bindings for exporting to analytics databases. The aggregator service maintains per-minute window state and the metrics API reads across windows to answer time-range queries. This architecture handles millions of events per minute without specialized streaming frameworks.
