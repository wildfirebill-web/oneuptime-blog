# How to Automate ClickHouse Performance Reporting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Performance, Reporting, Automation, Monitoring

Description: Automate ClickHouse performance reporting by querying system tables on a schedule and delivering key metrics to your team daily.

---

## Why Automated Reporting Matters

Ad-hoc performance reviews miss trends. Automated reports delivered to Slack or email give your team a consistent view of query latencies, disk usage, and merge backlogs before they become incidents.

## Key Metrics to Report

Focus on these system tables for a comprehensive performance snapshot:

- `system.query_log` - query durations and errors
- `system.merges` - active merge operations
- `system.parts` - part counts and sizes
- `system.disks` - disk utilization

## Top Slow Queries

Pull the top 10 slowest queries from the past 24 hours:

```sql
SELECT
    normalizeQuery(query) AS normalized,
    count() AS calls,
    avg(query_duration_ms) AS avg_ms,
    max(query_duration_ms) AS max_ms
FROM system.query_log
WHERE event_time >= now() - INTERVAL 1 DAY
  AND type = 'QueryFinish'
  AND query_duration_ms > 1000
GROUP BY normalized
ORDER BY avg_ms DESC
LIMIT 10;
```

## Merge Backlog Check

A large merge backlog causes read amplification. Report the current state:

```sql
SELECT
    database,
    table,
    count() AS active_merges,
    round(sum(rows_read) / 1e6, 2) AS total_rows_m
FROM system.merges
GROUP BY database, table
ORDER BY active_merges DESC;
```

## Disk Usage Summary

```sql
SELECT
    name AS disk,
    formatReadableSize(free_space) AS free,
    formatReadableSize(total_space) AS total,
    round((1 - free_space / total_space) * 100, 1) AS used_pct
FROM system.disks
ORDER BY used_pct DESC;
```

## Automating with a Cron Script

Create a script that runs these queries and emails the results:

```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d)
REPORT_FILE="/tmp/ch_report_${DATE}.txt"

clickhouse-client --query "
SELECT normalizeQuery(query), count(), avg(query_duration_ms)
FROM system.query_log
WHERE event_time >= now() - INTERVAL 1 DAY
  AND type = 'QueryFinish'
GROUP BY normalizeQuery(query)
ORDER BY avg(query_duration_ms) DESC
LIMIT 10
FORMAT TSV
" > "${REPORT_FILE}"

mail -s "ClickHouse Daily Performance Report ${DATE}" \
     team@example.com < "${REPORT_FILE}"
```

Schedule with cron:

```text
0 8 * * * /opt/scripts/ch_report.sh
```

## Pushing Metrics to OneUptime

For real-time visibility, push key metrics to OneUptime using its custom metrics API so dashboards stay current without waiting for a daily email.

```bash
SLOW_QUERIES=$(clickhouse-client --query \
  "SELECT count() FROM system.query_log WHERE query_duration_ms > 5000 AND event_time >= now() - INTERVAL 1 HOUR AND type = 'QueryFinish'" \
  --format TSVRaw)

curl -X POST https://oneuptime.example.com/api/ingest \
  -H "Content-Type: application/json" \
  -d "{\"metric\": \"clickhouse.slow_queries_1h\", \"value\": ${SLOW_QUERIES}}"
```

## Summary

Automating ClickHouse performance reporting through scheduled scripts that query system tables and deliver summaries to your team ensures performance issues are spotted early rather than after users complain.
