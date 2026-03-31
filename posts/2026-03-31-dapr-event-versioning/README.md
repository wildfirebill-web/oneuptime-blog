# How to Implement Event Versioning with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Event Sourcing, Versioning, State Management, Architecture

Description: Implement event versioning strategies in a Dapr event store to safely evolve event schemas over time without breaking existing event handlers or losing historical data.

---

Event schemas change as your domain model evolves. A field gets renamed, a new required field is added, or an event is split into two. Without versioning, schema changes break existing event handlers that read historical events. Dapr event stores need a clear versioning strategy from the start.

## Versioning Strategies

Three common approaches:

1. **Weak schema evolution**: Add optional fields, never remove or rename
2. **Event upcasting**: Transform old event format to new format when loading
3. **Explicit version field**: Include a version number in every event and handle each version explicitly

## Including Version in Every Event

Add a `version` field to your event schema:

```go
// event.go
type Event struct {
    EventID       string                 `json:"eventId"`
    EventType     string                 `json:"eventType"`
    Version       string                 `json:"version"`   // "1.0", "2.0"
    AggregateID   string                 `json:"aggregateId"`
    AggregateType string                 `json:"aggregateType"`
    Sequence      int64                  `json:"sequence"`
    Payload       map[string]interface{} `json:"payload"`
    OccurredAt    time.Time              `json:"occurredAt"`
}
```

## Upcasting Old Events

When loading events, transform old versions to the current format:

```go
// upcaster.go
func Upcast(event Event) Event {
    switch event.EventType {
    case "OrderCreated":
        return upcastOrderCreated(event)
    case "CustomerRegistered":
        return upcastCustomerRegistered(event)
    }
    return event
}

func upcastOrderCreated(event Event) Event {
    switch event.Version {
    case "1.0":
        // Version 1.0 used "price" instead of "total"
        if price, ok := event.Payload["price"]; ok {
            event.Payload["total"] = price
            delete(event.Payload, "price")
        }
        event.Version = "2.0"
    }
    return event
}
```

## Loading Events with Upcasting

Wrap the event loader to apply upcasting automatically:

```go
func LoadAndUpcastEvents(client dapr.Client, aggregateType, aggregateID string) ([]Event, error) {
    rawEvents, err := LoadEvents(client, aggregateType, aggregateID)
    if err != nil {
        return nil, err
    }

    upcasted := make([]Event, len(rawEvents))
    for i, e := range rawEvents {
        upcasted[i] = Upcast(e)
    }
    return upcasted, nil
}
```

## Publishing Versioned Events

Always publish the current version. Include the version in the event metadata:

```go
func PublishVersionedEvent(client dapr.Client, event Event) error {
    return client.PublishEvent(
        context.Background(),
        "pubsub",
        event.EventType,
        event,
        dapr.PublishEventWithMetadata(map[string]string{
            "version": event.Version,
        }),
    )
}
```

## Subscriber Version Handling

Subscribers should handle multiple versions or rely on the upcasting layer:

```go
func HandleOrderCreated(event Event) error {
    // Upcast ensures this always receives v2.0 format
    event = Upcast(event)

    // Now safe to access v2.0 fields
    total := event.Payload["total"].(float64)
    fmt.Printf("Order created with total: %.2f\n", total)
    return nil
}
```

## Adding New Required Fields Safely

When adding a required field to an event:
1. Deploy subscribers that handle the old schema (field absent)
2. Start publishing events with the new field
3. Write an upcaster that adds a default value for old events

```go
func upcastOrderCreated(event Event) Event {
    if event.Version == "2.0" {
        // Add currency field with default if missing
        if _, ok := event.Payload["currency"]; !ok {
            event.Payload["currency"] = "USD"
        }
        event.Version = "3.0"
    }
    return event
}
```

## Summary

Event versioning with Dapr requires combining a version field in every event, upcasting functions that transform old schemas to current format, and careful deployment ordering. By treating upcasting as a first-class concern, you can safely evolve event schemas without breaking existing handlers or losing the audit trail in historical events.
