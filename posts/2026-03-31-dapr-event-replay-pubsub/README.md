# How to Implement Event Replay with Dapr Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Event Replay, Event Sourcing, Recovery

Description: Learn how to implement event replay with Dapr pub/sub to reprocess historical events for bug fixes, new consumer bootstrapping, and projection rebuilds.

---

## Why Event Replay?

Event replay lets you reprocess past events without requiring the original services to re-execute business logic. Common uses: bootstrapping a new consumer with historical data, rebuilding a corrupted projection, or reprocessing events after a bug fix.

## Architecture for Replay

Keep events in a durable log (Kafka or a database) in addition to consuming them. A replay tool reads from the log and re-publishes to a replay topic that consumers can subscribe to during bootstrap.

## Storing Events for Replay

When consuming events, persist them to a replay log:

```javascript
const { DaprServer, DaprClient } = require('@dapr/dapr');
const server = new DaprServer();
const client = new DaprClient();

// Event archive consumer - runs alongside all other consumers
await server.pubsub.subscribe('events-pubsub', 'order.placed', async (event) => {
  await db.query(
    `INSERT INTO event_log (event_id, topic, payload, published_at)
     VALUES ($1, $2, $3, $4)
     ON CONFLICT (event_id) DO NOTHING`,
    [event.eventId, 'order.placed', JSON.stringify(event), new Date()]
  );
});
```

## Replay Tool

```javascript
async function replayEvents(topic, fromDate, toDate, replayTopic) {
  const client = new DaprClient();

  const events = await db.query(
    `SELECT payload FROM event_log
     WHERE topic = $1
       AND published_at BETWEEN $2 AND $3
     ORDER BY published_at ASC`,
    [topic, fromDate, toDate]
  );

  console.log(`Replaying ${events.rows.length} events to ${replayTopic}`);

  for (const row of events.rows) {
    const event = JSON.parse(row.payload);
    await client.pubsub.publish(
      'events-pubsub',
      replayTopic,
      { ...event, isReplay: true, replayedAt: new Date().toISOString() }
    );
    // Small delay to avoid overwhelming consumers
    await new Promise(r => setTimeout(r, 10));
  }

  console.log('Replay complete');
}
```

## Consumer Supporting Replay Mode

```javascript
const isReplay = process.env.REPLAY_MODE === 'true';

await server.pubsub.subscribe(
  'events-pubsub',
  isReplay ? 'order.placed.replay' : 'order.placed',
  async (event) => {
    if (event.isReplay) {
      console.log(`Processing replayed event from ${event.replayedAt}`);
    }
    await processOrderEvent(event);
  }
);
```

## Running a Targeted Replay

```bash
# Replay all order.placed events from last week to rebuild analytics
node replay-tool.js \
  --topic order.placed \
  --from "2026-03-24T00:00:00Z" \
  --to "2026-03-31T00:00:00Z" \
  --replay-topic order.placed.replay

# Start analytics service in replay mode
REPLAY_MODE=true node analytics-service.js
```

## Idempotency in Consumers

Replay consumers must be idempotent since events may be processed multiple times:

```javascript
async function processOrderEvent(event) {
  // Check if already processed
  const processed = await db.query(
    'SELECT 1 FROM processed_events WHERE event_id = $1',
    [event.eventId]
  );
  if (processed.rows.length > 0) return;

  // Process and mark as done atomically
  await db.transaction(async trx => {
    await processEvent(trx, event);
    await trx.query(
      'INSERT INTO processed_events (event_id) VALUES ($1)',
      [event.eventId]
    );
  });
}
```

## Summary

Event replay with Dapr pub/sub requires three components: a durable event log, a replay tool that re-publishes to dedicated topics, and idempotent consumers that tolerate duplicate delivery. This combination gives you the ability to recover from consumer bugs or bootstrap new services against historical data without re-running business transactions.
