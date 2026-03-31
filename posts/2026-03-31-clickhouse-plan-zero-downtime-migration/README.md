# How to Plan a Zero-Downtime Migration to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Migration, Zero Downtime, Database, Deployment, Strategy

Description: Plan and execute a zero-downtime migration to ClickHouse using dual-write, backfill, and cutover patterns that keep your application online throughout.

---

Migrating a production database with no downtime requires careful planning. This guide describes a proven dual-write approach to migrate any database to ClickHouse while keeping your application online.

## The Four Phases

1. **Prepare** - set up ClickHouse and validate schema
2. **Dual-write** - write to both old DB and ClickHouse simultaneously
3. **Backfill** - copy historical data in the background
4. **Cutover** - switch reads to ClickHouse and remove dual-write

## Phase 1 - Prepare

Create the ClickHouse schema matching your source:

```sql
CREATE TABLE events (
    event_id   UUID DEFAULT generateUUIDv4(),
    user_id    UInt64,
    event_type LowCardinality(String),
    created_at DateTime DEFAULT now(),
    value      Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id);
```

Validate with a small sample:

```sql
INSERT INTO events VALUES (generateUUIDv4(), 1, 'test', now(), 1.0);
SELECT * FROM events LIMIT 1;
```

## Phase 2 - Dual-Write

Update your application to write to both the old database and ClickHouse:

```python
def track_event(user_id: int, event_type: str, value: float):
    # Write to existing database
    existing_db.execute(
        "INSERT INTO events (user_id, event_type, value) VALUES (%s, %s, %s)",
        (user_id, event_type, value)
    )

    # Also write to ClickHouse (fire-and-forget with error capture)
    try:
        clickhouse_client.insert('events', [{
            'user_id': user_id,
            'event_type': event_type,
            'value': value,
        }])
    except Exception as e:
        logger.warning(f"ClickHouse dual-write failed: {e}")
        metrics.increment('clickhouse.dual_write.error')
```

Key principles:
- The old DB write must succeed first
- ClickHouse errors must never fail the request
- Use async writes or a queue (Kafka/Redis) for high throughput

## Phase 3 - Backfill Historical Data

Backfill data in time-range batches to avoid overloading either system:

```bash
#!/bin/bash
START="2023-01-01"
END="2024-01-01"
BATCH_DAYS=7

current="$START"
while [ "$current" < "$END" ]; do
  next=$(date -d "$current + $BATCH_DAYS days" +%Y-%m-%d)

  clickhouse-client --query "
    INSERT INTO events
    SELECT * FROM postgresql('old-db-host:5432', 'mydb', 'events', 'user', 'pass')
    WHERE created_at >= '$current' AND created_at < '$next'
  "

  echo "Backfilled $current to $next"
  current="$next"
  sleep 1
done
```

Monitor backfill progress:

```sql
SELECT
    toYYYYMM(created_at) AS month,
    count() AS rows,
    formatReadableSize(sum(data_compressed_bytes)) AS size
FROM system.parts
WHERE table = 'events' AND active
GROUP BY month
ORDER BY month;
```

## Phase 4 - Validate and Cutover

Before switching reads, validate data consistency:

```sql
-- Row counts per month
SELECT toYYYYMM(created_at) AS month, count() AS clickhouse_count
FROM events
GROUP BY month
ORDER BY month;

-- Compare with source (run the same query on the old DB)
```

Once validated, update your application to read from ClickHouse:

```python
ANALYTICS_READ_SOURCE = os.getenv('ANALYTICS_SOURCE', 'legacy')

def get_event_summary(from_date: str):
    if ANALYTICS_READ_SOURCE == 'clickhouse':
        return clickhouse_client.query(...)
    else:
        return legacy_db.query(...)
```

Gradually shift traffic using feature flags, then remove the dual-write code once confident.

## Rollback Plan

Keep the dual-write in place for at least one week after cutover. To roll back, set `ANALYTICS_SOURCE=legacy` and all reads revert instantly.

## Summary

A zero-downtime migration to ClickHouse follows four phases: prepare, dual-write, backfill, and cutover. The dual-write pattern ensures no data is lost during the transition, and the feature flag approach allows instant rollback if issues arise.
