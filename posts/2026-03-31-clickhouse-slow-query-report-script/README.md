# How to Write a ClickHouse Slow Query Report Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Script, Slow Query, Monitoring, Performance

Description: Write a script that extracts slow queries from ClickHouse's query log, formats them into a readable report, and helps identify optimization opportunities.

---

## Using system.query_log

ClickHouse logs completed queries to `system.query_log`. By querying this table, you can find the slowest queries over any time window without needing external APM tools.

## The Slow Query Report Script

```bash
#!/usr/bin/env bash
# clickhouse-slow-query-report.sh

CH_HOST="${CH_HOST:-localhost}"
CH_PORT="${CH_PORT:-8123}"
CH_USER="${CH_USER:-default}"
CH_PASSWORD="${CH_PASSWORD:-}"
SLOW_THRESHOLD_MS="${SLOW_THRESHOLD_MS:-5000}"
HOURS_BACK="${HOURS_BACK:-24}"

run_query() {
    curl -s "http://${CH_HOST}:${CH_PORT}/" \
        -u "${CH_USER}:${CH_PASSWORD}" \
        --data-binary "$1"
}

echo "=== ClickHouse Slow Query Report ==="
echo "Threshold: ${SLOW_THRESHOLD_MS}ms | Window: last ${HOURS_BACK} hours"
echo "Generated: $(date)"
echo ""

echo "--- Top 20 Slowest Queries ---"
run_query "
SELECT
    query_duration_ms,
    formatReadableQuantity(read_rows) AS rows_read,
    formatReadableSize(read_bytes) AS bytes_read,
    memory_usage,
    user,
    substring(query, 1, 200) AS query_preview
FROM system.query_log
WHERE
    type = 'QueryFinish'
    AND event_time >= now() - INTERVAL ${HOURS_BACK} HOUR
    AND query_duration_ms >= ${SLOW_THRESHOLD_MS}
    AND query NOT LIKE '%system.query_log%'
ORDER BY query_duration_ms DESC
LIMIT 20
FORMAT PrettyCompactMonoBlock
"

echo ""
echo "--- Slow Query Count by User ---"
run_query "
SELECT
    user,
    count() AS slow_query_count,
    avg(query_duration_ms) AS avg_ms,
    max(query_duration_ms) AS max_ms
FROM system.query_log
WHERE
    type = 'QueryFinish'
    AND event_time >= now() - INTERVAL ${HOURS_BACK} HOUR
    AND query_duration_ms >= ${SLOW_THRESHOLD_MS}
GROUP BY user
ORDER BY slow_query_count DESC
FORMAT PrettyCompactMonoBlock
"
```

## Finding Repeated Slow Query Patterns

Group by normalized query to find frequently repeated slow patterns:

```bash
run_query "
SELECT
    normalizeQuery(query) AS normalized,
    count() AS cnt,
    avg(query_duration_ms) AS avg_ms,
    max(query_duration_ms) AS max_ms
FROM system.query_log
WHERE
    type = 'QueryFinish'
    AND query_duration_ms >= ${SLOW_THRESHOLD_MS}
    AND event_time >= now() - INTERVAL 24 HOUR
GROUP BY normalized
ORDER BY cnt * avg_ms DESC
LIMIT 10
FORMAT PrettyCompactMonoBlock
"
```

## Checking Index Usage for Slow Queries

```bash
run_query "
SELECT
    query_id,
    query_duration_ms,
    ProfileEvents['SelectedParts'] AS parts_selected,
    ProfileEvents['SelectedMarks'] AS marks_selected,
    ProfileEvents['SelectedRanges'] AS ranges_selected,
    substring(query, 1, 150) AS query_preview
FROM system.query_log
WHERE
    type = 'QueryFinish'
    AND query_duration_ms >= ${SLOW_THRESHOLD_MS}
ORDER BY query_duration_ms DESC
LIMIT 10
FORMAT PrettyCompactMonoBlock
"
```

High `marks_selected` relative to total marks indicates poor index selectivity.

## Scheduling the Report

```bash
# Run every morning at 7 AM and email the report
echo "0 7 * * * CH_HOST=ch-prod /opt/scripts/clickhouse-slow-query-report.sh | mail -s 'ClickHouse Slow Queries' team@example.com" | crontab -
```

## Summary

The ClickHouse slow query report script queries `system.query_log` to surface the slowest queries, group them by user and normalized pattern, and show index usage metrics. Scheduling it daily creates a feedback loop for continuous query optimization.
