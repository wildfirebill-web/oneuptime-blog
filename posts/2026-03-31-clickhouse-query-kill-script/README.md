# How to Write a ClickHouse Query Kill Script for Long Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Script, KILL QUERY, Administration, Operation

Description: Write a script that identifies and kills long-running ClickHouse queries based on configurable time thresholds, with dry-run and user filtering options.

---

## When Queries Need to Be Killed

Runaway queries in ClickHouse can consume all memory, saturate CPU, or hold up background merges. While ClickHouse has per-user `max_execution_time` settings, you sometimes need an emergency kill script to clean up queries that escaped those limits.

## The Query Kill Script

```bash
#!/usr/bin/env bash
# clickhouse-query-kill.sh

CH_HOST="${CH_HOST:-localhost}"
CH_PORT="${CH_PORT:-8123}"
CH_USER="${CH_USER:-default}"
CH_PASSWORD="${CH_PASSWORD:-}"
MAX_ELAPSED="${MAX_ELAPSED:-300}"   # kill queries running longer than 5 min
DRY_RUN="${DRY_RUN:-1}"            # set to 0 to actually kill
EXCLUDE_USER="${EXCLUDE_USER:-system}"

run_query() {
    curl -s "http://${CH_HOST}:${CH_PORT}/" \
        -u "${CH_USER}:${CH_PASSWORD}" \
        --data-binary "$1"
}

echo "=== ClickHouse Query Kill Script ==="
echo "Max elapsed: ${MAX_ELAPSED}s | Dry run: ${DRY_RUN} | Exclude user: ${EXCLUDE_USER}"
echo ""

echo "--- Currently Running Queries ---"
run_query "
SELECT
    query_id,
    user,
    elapsed,
    formatReadableSize(memory_usage) AS memory,
    substring(query, 1, 120) AS query_preview
FROM system.processes
WHERE elapsed > 10
ORDER BY elapsed DESC
FORMAT PrettyCompactMonoBlock
"

echo ""
echo "--- Queries to Kill (elapsed > ${MAX_ELAPSED}s) ---"

QUERY_IDS=$(run_query "
SELECT query_id
FROM system.processes
WHERE elapsed > ${MAX_ELAPSED}
  AND user != '${EXCLUDE_USER}'
FORMAT TabSeparated
")

if [ -z "$QUERY_IDS" ]; then
    echo "  None - no long-running queries found"
    exit 0
fi

echo "$QUERY_IDS"

if [ "$DRY_RUN" = "0" ]; then
    echo ""
    echo "--- Killing queries... ---"
    while IFS= read -r query_id; do
        if [ -n "$query_id" ]; then
            echo "  Killing: ${query_id}"
            run_query "KILL QUERY WHERE query_id = '${query_id}' SYNC"
        fi
    done <<< "$QUERY_IDS"
    echo "Done."
else
    echo ""
    echo "DRY RUN - set DRY_RUN=0 to actually kill these queries"
fi
```

## Killing by User

To kill all queries from a specific user:

```bash
run_query "KILL QUERY WHERE user = 'batch_user' SYNC"
```

## Killing Mutations

Long-running mutations can also block operations. Kill them with:

```bash
run_query "
KILL MUTATION WHERE table = 'events' AND mutation_id = 'mutation_5.txt'
"
```

## Setting Automatic Limits

Rather than relying on kill scripts, set limits at the profile level:

```sql
ALTER USER analyst SETTINGS max_execution_time = 120;  -- 2 minute timeout
```

## Summary

The query kill script lists processes from `system.processes`, filters by elapsed time and excluded users, and issues `KILL QUERY` commands in sync mode to immediately stop offending queries. A dry-run mode lets you preview what would be killed before committing.
