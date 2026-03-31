# How to Use Dapr Pub/Sub for Event Sourcing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Event Sourcing, Architecture, Microservice

Description: Learn how to implement event sourcing patterns using Dapr pub/sub and state management to build append-only event logs with reproducible state.

---

## Event Sourcing with Dapr

In event sourcing, the current state of an entity is derived by replaying an ordered sequence of immutable events rather than storing mutable records. Dapr provides the building blocks for event sourcing:

- **Pub/Sub**: Event bus for distributing domain events
- **State Management**: Storing the event log and current projections
- **Actors**: Optional stateful aggregate hosts

## Architecture Overview

The event sourcing flow in Dapr:
1. A command arrives at the aggregate service
2. The aggregate validates the command and emits domain events
3. Events are appended to the event log (state store)
4. Events are published to a Dapr topic for downstream consumers
5. Projections rebuild their read models from the event stream

## Storing Events to the Event Log

Each domain event is appended to an ordered log using a versioned key:

```javascript
const { DaprClient } = require("@dapr/dapr");
const client = new DaprClient();

async function appendEvent(aggregateId, version, event) {
  const eventKey = `${aggregateId}:event:${version.toString().padStart(10, "0")}`;

  await client.state.save("event-store", [
    {
      key: eventKey,
      value: {
        aggregateId,
        version,
        type: event.type,
        data: event.data,
        timestamp: new Date().toISOString(),
      },
    },
  ]);

  return eventKey;
}
```

## Publishing Events After Append

After persisting to the event log, publish the event for downstream projections:

```javascript
async function handlePlaceOrderCommand(command) {
  const aggregateId = command.orderId;

  // Load current version
  const meta = await client.state.get("event-store", `${aggregateId}:meta`);
  const currentVersion = meta?.version ?? 0;
  const nextVersion = currentVersion + 1;

  const event = {
    type: "OrderPlaced",
    data: {
      orderId: aggregateId,
      customerId: command.customerId,
      items: command.items,
      total: command.total,
    },
  };

  // Append to event log
  await appendEvent(aggregateId, nextVersion, event);

  // Update metadata
  await client.state.save("event-store", [
    {
      key: `${aggregateId}:meta`,
      value: { version: nextVersion, updatedAt: new Date().toISOString() },
    },
  ]);

  // Publish for projections
  await client.pubsub.publish("pubsub", "domain-events", {
    aggregateId,
    version: nextVersion,
    ...event,
  });
}
```

## Building a Projection from the Event Stream

A projection service subscribes to `domain-events` and maintains a read-optimized view:

```javascript
app.get("/dapr/subscribe", (req, res) => {
  res.json([
    {
      pubsubname: "pubsub",
      topic: "domain-events",
      route: "/projections/orders",
    },
  ]);
});

app.post("/projections/orders", async (req, res) => {
  const event = req.body.data;

  if (event.type === "OrderPlaced") {
    await client.state.save("query-store", [
      {
        key: `order:${event.aggregateId}`,
        value: {
          orderId: event.aggregateId,
          status: "placed",
          total: event.data.total,
          version: event.version,
        },
      },
    ]);
  }

  if (event.type === "OrderShipped") {
    const current = await client.state.get(
      "query-store",
      `order:${event.aggregateId}`
    );
    await client.state.save("query-store", [
      {
        key: `order:${event.aggregateId}`,
        value: { ...current, status: "shipped", version: event.version },
      },
    ]);
  }

  res.sendStatus(200);
});
```

## Replaying Events to Rebuild State

To rebuild a projection, iterate the event log keys in version order:

```bash
# Using Dapr state query API to fetch events for an aggregate
curl -s -X POST \
  http://localhost:3500/v1.0-alpha1/state/event-store/query \
  -H "Content-Type: application/json" \
  -d '{
    "filter": {
      "EQ": { "aggregateId": "ord-123" }
    },
    "sort": [{"key": "version", "order": "ASC"}]
  }'
```

## Summary

Dapr pub/sub combined with state management enables event sourcing by providing an event bus for distributing domain events and a persistent store for the append-only event log. Publishing events after append ensures projections stay in sync, and the event log can be replayed to rebuild any projection from scratch - giving you full auditability and temporal queries.
