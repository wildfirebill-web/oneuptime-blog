# How to Implement Event Sourcing with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Event Sourcing, Analytics, Architecture, Append-Only

Description: Learn how to use ClickHouse as an append-only event store for event sourcing patterns, with materialized views for real-time projections.

---

## Event Sourcing Meets ClickHouse

Event sourcing stores every state change as an immutable event. ClickHouse is naturally append-only, making it an excellent event store for analytics-oriented event sourcing. It excels at reading history fast and building projections over millions of events.

## Event Log Table

Create the central event log with a flexible schema:

```sql
CREATE TABLE domain_events
(
    event_id        UUID DEFAULT generateUUIDv4(),
    aggregate_id    String,
    aggregate_type  LowCardinality(String),
    event_type      LowCardinality(String),
    event_time      DateTime64(3),
    sequence_num    UInt64,
    payload         String,
    metadata        String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (aggregate_type, aggregate_id, sequence_num);
```

The `sequence_num` within an aggregate guarantees ordering for replays.

## Publishing Events

Insert events as they are emitted:

```sql
INSERT INTO domain_events
(aggregate_id, aggregate_type, event_type, event_time, sequence_num, payload)
VALUES
('order-123', 'Order', 'OrderPlaced', now64(), 1, '{"amount": 99.99}'),
('order-123', 'Order', 'PaymentReceived', now64(), 2, '{"method": "card"}');
```

## Replaying Events for an Aggregate

Fetch the full history of an aggregate in order:

```sql
SELECT event_type, event_time, sequence_num, payload
FROM domain_events
WHERE aggregate_type = 'Order'
  AND aggregate_id = 'order-123'
ORDER BY sequence_num;
```

## Materializing Projections

Use materialized views to build read models that update as events arrive:

```sql
CREATE MATERIALIZED VIEW order_totals_mv
ENGINE = SummingMergeTree()
ORDER BY aggregate_id
AS
SELECT
    aggregate_id,
    countIf(event_type = 'OrderPlaced') AS order_count,
    sumIf(JSONExtractFloat(payload, 'amount'), event_type = 'OrderPlaced') AS total_revenue
FROM domain_events
GROUP BY aggregate_id;
```

## Snapshotting

For aggregates with long histories, store snapshots to speed up replay:

```sql
CREATE TABLE aggregate_snapshots
(
    aggregate_id    String,
    aggregate_type  LowCardinality(String),
    sequence_num    UInt64,
    snapshot_time   DateTime,
    state           String
)
ENGINE = ReplacingMergeTree(sequence_num)
ORDER BY (aggregate_type, aggregate_id);
```

Replay only events after the latest snapshot `sequence_num`.

## Querying Event Streams

Analyze event patterns across all aggregates:

```sql
SELECT
    event_type,
    count() AS occurrences,
    avg(sequence_num) AS avg_sequence
FROM domain_events
WHERE event_time >= today() - 30
GROUP BY event_type
ORDER BY occurrences DESC;
```

## Summary

ClickHouse is a natural fit for event sourcing because of its append-only nature, fast sequential reads, and powerful materialized views. Store events in a partitioned MergeTree table, replay aggregate histories via sorted queries, and build read projections with materialized views for low-latency analytics.
