# How to Write a ClickHouse Disk Usage Alert Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Disk Usage, Script, Alert, Monitoring

Description: Build a ClickHouse disk usage alert script that monitors all configured disks, warns at thresholds, and reports which tables are the largest consumers.

---

## Disk Management in ClickHouse

ClickHouse can use multiple disks configured in a storage policy. Each disk can fill independently. Without active monitoring, a single fast-growing table can fill a disk and cause all writes to fail.

## The Disk Usage Alert Script

```bash
#!/usr/bin/env bash
# clickhouse-disk-usage-alert.sh

CH_HOST="${CH_HOST:-localhost}"
CH_PORT="${CH_PORT:-8123}"
CH_USER="${CH_USER:-default}"
CH_PASSWORD="${CH_PASSWORD:-}"
WARN_THRESHOLD="${WARN_THRESHOLD:-75}"    # percent
CRIT_THRESHOLD="${CRIT_THRESHOLD:-90}"   # percent

run_query() {
    curl -s "http://${CH_HOST}:${CH_PORT}/" \
        -u "${CH_USER}:${CH_PASSWORD}" \
        --data-binary "$1"
}

EXIT_CODE=0

echo "=== ClickHouse Disk Usage Alert - $(date) ==="
echo ""

echo "--- Disk Usage Overview ---"
run_query "
SELECT
    name AS disk_name,
    path,
    formatReadableSize(total_space) AS total,
    formatReadableSize(free_space) AS free,
    formatReadableSize(total_space - free_space) AS used,
    round((total_space - free_space) * 100.0 / total_space, 1) AS used_pct
FROM system.disks
ORDER BY used_pct DESC
FORMAT PrettyCompactMonoBlock
"

echo ""
echo "--- Checking Thresholds ---"
while IFS=$'\t' read -r disk_name used_pct; do
    used_int=${used_pct%.*}
    if [ "$used_int" -ge "$CRIT_THRESHOLD" ]; then
        echo "  CRITICAL: Disk '${disk_name}' is ${used_pct}% full"
        EXIT_CODE=2
    elif [ "$used_int" -ge "$WARN_THRESHOLD" ]; then
        echo "  WARNING: Disk '${disk_name}' is ${used_pct}% full"
        [ $EXIT_CODE -lt 1 ] && EXIT_CODE=1
    else
        echo "  OK: Disk '${disk_name}' is ${used_pct}% full"
    fi
done < <(run_query "
SELECT
    name,
    toString(round((total_space - free_space) * 100.0 / total_space, 0))
FROM system.disks
FORMAT TabSeparated
")

echo ""
echo "--- Top 10 Tables by Disk Usage ---"
run_query "
SELECT
    database,
    table,
    disk_name,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE active = 1
GROUP BY database, table, disk_name
ORDER BY sum(bytes_on_disk) DESC
LIMIT 10
FORMAT PrettyCompactMonoBlock
"

echo ""
echo "--- ClickHouse Data Directory Usage ---"
run_query "
SELECT
    formatReadableSize(sum(bytes_on_disk)) AS total_ch_data_size,
    count(DISTINCT database || '.' || table) AS table_count,
    sum(rows) AS total_rows
FROM system.parts
WHERE active = 1
FORMAT PrettyCompactMonoBlock
"

exit $EXIT_CODE
```

## Scheduling Alerts

```bash
# Check every hour
echo "0 * * * * /opt/scripts/clickhouse-disk-usage-alert.sh || \
  /opt/scripts/clickhouse-disk-usage-alert.sh 2>&1 | \
  mail -s 'ClickHouse Disk Alert' ops@example.com" | crontab -
```

## Summary

The disk usage alert script queries `system.disks` and `system.parts` to monitor disk utilization across all ClickHouse storage volumes, emit warnings and critical alerts at configurable thresholds, and report the largest tables so you know where to apply TTL or partition cleanup.
