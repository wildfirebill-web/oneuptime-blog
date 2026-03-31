# How to Write a ClickHouse Replication Status Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Replication, Script, Monitoring, High Availability

Description: Write a script that checks ClickHouse replication lag, queue size, and replica health across all replicated tables and alerts when replicas fall behind.

---

## Why Monitor Replication Separately?

ClickHouse replication lag is not always visible through standard server health checks. A replica can be "up" while lagging minutes or hours behind the primary. This script makes replication health visible at a glance.

## The Replication Status Script

```bash
#!/usr/bin/env bash
# clickhouse-replication-status.sh

CH_HOST="${CH_HOST:-localhost}"
CH_PORT="${CH_PORT:-8123}"
CH_USER="${CH_USER:-default}"
CH_PASSWORD="${CH_PASSWORD:-}"
LAG_THRESHOLD="${LAG_THRESHOLD:-60}"    # seconds
QUEUE_THRESHOLD="${QUEUE_THRESHOLD:-50}"

run_query() {
    curl -s "http://${CH_HOST}:${CH_PORT}/" \
        -u "${CH_USER}:${CH_PASSWORD}" \
        --data-binary "$1"
}

ALERT=0

echo "=== ClickHouse Replication Status - $(date) ==="
echo ""

echo "--- Replication Overview ---"
run_query "
SELECT
    database,
    table,
    replica_name,
    is_leader,
    is_readonly,
    is_session_expired,
    future_parts,
    parts_to_check,
    queue_size,
    absolute_delay,
    total_replicas,
    active_replicas
FROM system.replicas
ORDER BY absolute_delay DESC
FORMAT PrettyCompactMonoBlock
"

echo ""
echo "--- Replicas with High Lag (>${LAG_THRESHOLD}s) ---"
LAGGING=$(run_query "
SELECT
    database,
    table,
    replica_name,
    absolute_delay AS lag_seconds
FROM system.replicas
WHERE absolute_delay > ${LAG_THRESHOLD}
ORDER BY absolute_delay DESC
FORMAT TabSeparated
")

if [ -n "$LAGGING" ]; then
    echo "$LAGGING"
    ALERT=1
else
    echo "  None - all replicas are in sync"
fi

echo ""
echo "--- Replicas with Large Queue (>${QUEUE_THRESHOLD}) ---"
QUEUED=$(run_query "
SELECT
    database,
    table,
    replica_name,
    queue_size
FROM system.replicas
WHERE queue_size > ${QUEUE_THRESHOLD}
ORDER BY queue_size DESC
FORMAT TabSeparated
")

if [ -n "$QUEUED" ]; then
    echo "$QUEUED"
    ALERT=1
else
    echo "  None - all queues are small"
fi

echo ""
echo "--- Replication Queue Details (top 10 oldest entries) ---"
run_query "
SELECT
    database,
    table,
    type,
    create_time,
    dateDiff('second', create_time, now()) AS age_seconds,
    parts_to_merge,
    source_replica,
    is_currently_executing
FROM system.replication_queue
ORDER BY create_time
LIMIT 10
FORMAT PrettyCompactMonoBlock
"

if [ $ALERT -eq 1 ]; then
    echo ""
    echo "WARNING: Replication issues detected!"
    exit 1
fi

echo ""
echo "All replication checks passed."
exit 0
```

## Sending Alerts

```bash
# Pipe output to alerting on failure
./clickhouse-replication-status.sh || \
    ./clickhouse-replication-status.sh 2>&1 | \
    mail -s "ALERT: ClickHouse Replication Issues" ops@example.com
```

## Summary

The replication status script monitors `system.replicas` and `system.replication_queue` for lag, queue depth, and session health. It exits with code 1 when thresholds are breached, making it easy to plug into any alerting pipeline.
