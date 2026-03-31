# How to Query Event History with Dapr State Query API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Event Sourcing, State Query, API, Architecture

Description: Use the Dapr state query API to filter and retrieve event history from your event store by aggregate type, event type, time range, or custom metadata fields.

---

Reading a specific aggregate's event stream by iterating sequence numbers is efficient, but sometimes you need cross-aggregate queries: all events of a certain type in the last hour, all events for a specific customer, or events in a specific time range. The Dapr state query API enables these queries without a separate query database.

## Dapr State Query API Overview

The state query API lets you filter state store entries using JSON query syntax. Not all state store backends support it - PostgreSQL, MongoDB, and CosmosDB do.

```bash
POST http://localhost:3500/v1.0-alpha1/state/{storeName}/query
```

## Query Events by Event Type

```javascript
// query-events.js
const axios = require('axios');

async function queryEventsByType(eventType) {
  const query = {
    filter: {
      EQ: { "eventType": eventType }
    },
    sort: [
      { key: "occurredAt", order: "ASC" }
    ],
    page: {
      limit: 100
    }
  };

  const resp = await axios.post(
    'http://localhost:3500/v1.0-alpha1/state/event-store/query',
    query,
    { headers: { 'Content-Type': 'application/json' } }
  );

  return resp.data.results;
}
```

## Query Events by Aggregate

```javascript
async function queryAggregateEvents(aggregateType, aggregateId) {
  const query = {
    filter: {
      AND: [
        { EQ: { "aggregateType": aggregateType } },
        { EQ: { "aggregateId": aggregateId } }
      ]
    },
    sort: [{ key: "sequence", order: "ASC" }],
    page: { limit: 500 }
  };

  const resp = await axios.post(
    'http://localhost:3500/v1.0-alpha1/state/event-store/query',
    query
  );

  return resp.data.results.map(r => r.data);
}
```

## Query Events in a Time Range

```javascript
async function queryEventsInRange(fromDate, toDate) {
  const query = {
    filter: {
      AND: [
        { GT: { "occurredAt": fromDate.toISOString() } },
        { LT: { "occurredAt": toDate.toISOString() } }
      ]
    },
    sort: [{ key: "occurredAt", order: "ASC" }],
    page: { limit: 1000 }
  };

  const resp = await axios.post(
    'http://localhost:3500/v1.0-alpha1/state/event-store/query',
    query
  );

  return resp.data.results;
}
```

## Pagination for Large Result Sets

```javascript
async function queryAllEventsPaged(eventType) {
  const allEvents = [];
  let token = null;

  do {
    const query = {
      filter: { EQ: { "eventType": eventType } },
      sort: [{ key: "occurredAt", order: "ASC" }],
      page: {
        limit: 100,
        ...(token && { token })
      }
    };

    const resp = await axios.post(
      'http://localhost:3500/v1.0-alpha1/state/event-store/query',
      query
    );

    allEvents.push(...resp.data.results);
    token = resp.data.token;
  } while (token);

  return allEvents;
}
```

## State Store Requirements

Enable JSON querying in your state store component:

```yaml
# For MongoDB
spec:
  type: state.mongodb
  metadata:
    - name: operationTimeout
      value: "5s"

# For PostgreSQL, ensure JSONB is used (default)
spec:
  type: state.postgresql
  metadata:
    - name: connectionString
      value: "..."
```

## Summary

The Dapr state query API provides flexible event history querying without requiring a dedicated query database. By storing event metadata as structured JSON fields, you can filter by event type, aggregate, time range, or any custom metadata field - making cross-aggregate queries practical in Dapr event stores.
