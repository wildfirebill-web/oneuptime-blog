# How to Write a ClickHouse Data Retention Cleanup Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Data Retention, Script, TTL, Administration

Description: Build a data retention cleanup script for ClickHouse that drops old partitions and validates TTL execution, supplementing ClickHouse's built-in TTL with explicit control.

---

## TTL vs. Manual Cleanup

ClickHouse TTL automatically deletes old data during background merges. However, TTL execution can lag by hours or days depending on merge activity. For compliance or storage-pressure situations, a manual cleanup script gives immediate control.

## The Data Retention Cleanup Script

```bash
#!/usr/bin/env bash
# clickhouse-retention-cleanup.sh

CH_HOST="${CH_HOST:-localhost}"
CH_PORT="${CH_PORT:-8123}"
CH_USER="${CH_USER:-default}"
CH_PASSWORD="${CH_PASSWORD:-}"

# Define retention policies per table (days)
declare -A RETENTION_DAYS=(
    ["default.events"]="90"
    ["default.logs"]="30"
    ["default.metrics"]="365"
)

run_query() {
    curl -s "http://${CH_HOST}:${CH_PORT}/" \
        -u "${CH_USER}:${CH_PASSWORD}" \
        --data-binary "$1"
}

echo "=== ClickHouse Data Retention Cleanup - $(date) ==="
echo ""

for table_full in "${!RETENTION_DAYS[@]}"; do
    DAYS="${RETENTION_DAYS[$table_full]}"
    DB="${table_full%%.*}"
    TABLE="${table_full##*.}"
    CUTOFF=$(date -d "-${DAYS} days" +%Y%m%d)

    echo "--- Processing ${table_full} (retain ${DAYS} days) ---"

    # Find partitions older than cutoff
    OLD_PARTITIONS=$(run_query "
    SELECT DISTINCT partition
    FROM system.parts
    WHERE database = '${DB}' AND table = '${TABLE}'
      AND active = 1
      AND partition < '${CUTOFF}'
    ORDER BY partition
    FORMAT TabSeparated
    ")

    if [ -z "$OLD_PARTITIONS" ]; then
        echo "  No old partitions found for ${table_full}"
        continue
    fi

    echo "  Partitions to drop: $(echo "$OLD_PARTITIONS" | wc -l)"
    echo "$OLD_PARTITIONS"

    while IFS= read -r partition; do
        if [ -n "$partition" ]; then
            echo "  Dropping partition ${partition}..."
            RESULT=$(run_query "ALTER TABLE ${table_full} DROP PARTITION '${partition}'")
            if [ -n "$RESULT" ]; then
                echo "  ERROR: ${RESULT}"
            else
                echo "  Dropped: ${partition}"
            fi
        fi
    done <<< "$OLD_PARTITIONS"

    echo ""
done

echo "--- Final Storage Report ---"
run_query "
SELECT
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    min(min_date) AS oldest_data
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC
FORMAT PrettyCompactMonoBlock
"
```

## Triggering TTL Manually

If you prefer TTL but need immediate execution:

```sql
ALTER TABLE events MATERIALIZE TTL;
```

This forces ClickHouse to apply TTL rules immediately rather than waiting for the next background merge.

## Verifying Retention

After cleanup, confirm the oldest data:

```sql
SELECT min(event_date), max(event_date), count() FROM events;
```

## Summary

A data retention cleanup script explicitly drops partitions older than configurable thresholds for each table, providing immediate control over data lifecycle. Combined with `MATERIALIZE TTL` for TTL-based tables, it ensures compliance with data retention policies without waiting for background merges.
