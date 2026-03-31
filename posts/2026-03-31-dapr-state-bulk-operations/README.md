# How to Use Bulk State Operations in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, State Management, Bulk, Performance, Throughput

Description: Learn how to use Dapr bulk state operations to save, get, and delete multiple state keys in a single API call, improving throughput and reducing round trips.

---

## Why Use Bulk Operations?

Bulk state operations let you read or write multiple keys in a single API call, reducing network round trips and improving throughput. Instead of making 100 individual HTTP requests to save 100 session records, you make one.

## Bulk Save (Multiple Keys)

```bash
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[
    {"key": "user:1", "value": {"name": "Alice", "score": 100}},
    {"key": "user:2", "value": {"name": "Bob",   "score": 200}},
    {"key": "user:3", "value": {"name": "Carol",  "score": 150}},
    {"key": "user:4", "value": {"name": "Dave",   "score": 175}}
  ]'
```

## Bulk Get

Retrieve multiple keys in a single call using the bulk get API:

```bash
curl -X POST http://localhost:3500/v1.0/state/statestore/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "keys": ["user:1", "user:2", "user:3", "user:4"],
    "parallelism": 4
  }'
```

Response:

```json
[
  {"key": "user:1", "data": {"name": "Alice", "score": 100}, "etag": "v1"},
  {"key": "user:2", "data": {"name": "Bob",   "score": 200}, "etag": "v2"},
  {"key": "user:3", "data": {"name": "Carol",  "score": 150}, "etag": "v3"},
  {"key": "user:4", "error": ""}
]
```

Missing keys return an empty `data` field without an error.

## Bulk Get in Node.js SDK

```javascript
const { DaprClient } = require('@dapr/dapr');
const client = new DaprClient();

const results = await client.state.getBulk('statestore', [
  'user:1', 'user:2', 'user:3', 'user:4'
]);

results.forEach(item => {
  if (item.data) {
    console.log(`${item.key}: ${JSON.stringify(item.data)}`);
  } else {
    console.log(`${item.key}: not found`);
  }
});
```

## Bulk Save in the Python SDK

```python
from dapr.clients import DaprClient
from dapr.clients.grpc._state import StateItem

with DaprClient() as client:
    client.save_bulk_state(
        store_name='statestore',
        states=[
            StateItem(key='user:1', value='{"name": "Alice"}'),
            StateItem(key='user:2', value='{"name": "Bob"}'),
            StateItem(key='user:3', value='{"name": "Carol"}'),
        ]
    )
```

## Bulk Delete

Delete multiple keys with a single transaction:

```bash
curl -X POST http://localhost:3500/v1.0/state/statestore/transaction \
  -H "Content-Type: application/json" \
  -d '{
    "operations": [
      {"operation": "delete", "request": {"key": "user:1"}},
      {"operation": "delete", "request": {"key": "user:2"}},
      {"operation": "delete", "request": {"key": "user:3"}}
    ]
  }'
```

## Performance Considerations

- Bulk get `parallelism` controls how many keys are fetched concurrently from the backend
- Increase parallelism for state stores that support concurrent reads (Redis, Cosmos DB)
- Decrease parallelism if the state store is overwhelmed

## Summary

Dapr bulk state operations allow saving multiple keys with a single POST to `/v1.0/state/{store}` and retrieving multiple keys with a POST to `/v1.0/state/{store}/bulk`. Bulk operations significantly reduce network round trips for workloads that need to read or write many keys, such as session management, leaderboard updates, or batch data processing.
