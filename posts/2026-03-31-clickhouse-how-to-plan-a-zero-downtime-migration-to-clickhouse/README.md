# How to Plan a Zero-Downtime Migration to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Migration, Zero-Downtime, Strategy, ETL, Data Engineering

Description: Plan and execute a zero-downtime migration to ClickHouse using dual-write, incremental backfill, and cutover strategies to avoid service interruptions.

---

Migrating a production database to ClickHouse without downtime requires careful planning. The core strategy is dual-write: write to both the old system and ClickHouse simultaneously while backfilling historical data, then cut over reads when parity is confirmed.

## Migration Phases

```text
Phase 1: Provision ClickHouse
Phase 2: Dual-write (old system + ClickHouse)
Phase 3: Historical backfill
Phase 4: Validate parity
Phase 5: Shift reads to ClickHouse
Phase 6: Decommission old system
```

## Phase 1 - Provision ClickHouse and Schema

Create the destination table before touching production traffic:

```sql
CREATE TABLE analytics.events
(
    event_id    String,
    user_id     String,
    event_type  LowCardinality(String),
    created_at  DateTime,
    payload     String DEFAULT ''
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id, event_id);
```

## Phase 2 - Dual-Write in the Application

Update your application to write to both systems:

```python
def record_event(event: dict):
    # Write to existing system
    existing_db.insert(event)

    # Write to ClickHouse (fire and forget, don't fail the request)
    try:
        clickhouse_client.insert('analytics.events', [event])
    except Exception as e:
        logger.error(f"ClickHouse write failed: {e}")
        metrics.increment('clickhouse.write.error')
```

Make ClickHouse writes non-blocking so failures don't affect existing functionality.

## Phase 3 - Historical Backfill

Backfill historical data in time-ordered chunks to avoid overwhelming ClickHouse:

```python
import clickhouse_connect
from datetime import date, timedelta

client = clickhouse_connect.get_client(host='localhost', port=8123)

start = date(2023, 1, 1)
end = date.today()
chunk = timedelta(days=7)

current = start
while current < end:
    next_chunk = min(current + chunk, end)
    rows = old_db.query(
        "SELECT * FROM events WHERE created_at >= %s AND created_at < %s",
        (current, next_chunk)
    )
    if rows:
        client.insert('analytics.events', rows)
        print(f"Backfilled {len(rows)} rows for {current} to {next_chunk}")
    current = next_chunk
```

## Phase 4 - Validate Data Parity

Compare aggregate counts between old system and ClickHouse:

```sql
-- Run on ClickHouse
SELECT
    toDate(created_at) AS day,
    count() AS clickhouse_count
FROM analytics.events
WHERE created_at >= today() - 7
GROUP BY day
ORDER BY day;
```

```sql
-- Run on old system (adjust syntax as needed)
SELECT
    DATE(created_at) AS day,
    COUNT(*) AS old_count
FROM events
WHERE created_at >= NOW() - INTERVAL 7 DAY
GROUP BY day
ORDER BY day;
```

Automate this comparison:

```python
def check_parity(date: str) -> bool:
    ch_count = clickhouse_client.query(
        f"SELECT count() FROM analytics.events WHERE toDate(created_at) = '{date}'"
    ).result_rows[0][0]

    old_count = old_db.query_value(
        f"SELECT COUNT(*) FROM events WHERE DATE(created_at) = '{date}'"
    )

    diff_pct = abs(ch_count - old_count) / max(old_count, 1) * 100
    return diff_pct < 0.01  # Accept up to 0.01% variance
```

## Phase 5 - Cut Over Reads

Use a feature flag to gradually shift read traffic:

```python
def get_events(user_id: str, days: int):
    if feature_flags.is_enabled('clickhouse_reads', percentage=10):
        return clickhouse_client.query(
            "SELECT * FROM analytics.events WHERE user_id = {uid:String} LIMIT 100",
            parameters={'uid': user_id}
        ).result_rows
    else:
        return old_db.query("SELECT * FROM events WHERE user_id = %s LIMIT 100", (user_id,))
```

Increase the percentage from 10% to 100% over several days while monitoring error rates.

## Phase 6 - Decommission

Once reads are fully on ClickHouse and stable for 1-2 weeks:
1. Remove dual-write code
2. Archive historical data from the old system
3. Scale down or decommission the old system

## Summary

Zero-downtime migration to ClickHouse follows a predictable pattern: provision the new system, dual-write, backfill, validate, then cut over reads gradually. The key is making ClickHouse writes non-blocking during the dual-write phase and automating parity checks to catch discrepancies before cutover.
