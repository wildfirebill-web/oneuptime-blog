# How to Write a ClickHouse Health Check Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Health Check, Script, Monitoring, Operations

Description: Write a comprehensive ClickHouse health check script that validates server responsiveness, replication status, part counts, and disk usage in one go.

---

## What to Check in a Health Check

A useful ClickHouse health check should verify:

1. The server responds to queries (liveness)
2. No tables are in a broken replication state
3. Part counts are within safe limits
4. Disk usage is not critical
5. No long-running queries are blocking resources

## The Health Check Script

```bash
#!/usr/bin/env bash
# clickhouse-health-check.sh

CH_HOST="${CH_HOST:-localhost}"
CH_PORT="${CH_PORT:-8123}"
CH_USER="${CH_USER:-default}"
CH_PASSWORD="${CH_PASSWORD:-}"

ERRORS=0
WARNINGS=0

run_query() {
    curl -s --max-time 10 \
        "http://${CH_HOST}:${CH_PORT}/" \
        -u "${CH_USER}:${CH_PASSWORD}" \
        --data-binary "$1"
}

check_liveness() {
    echo "[CHECK] Server liveness..."
    RESULT=$(run_query "SELECT 1 FORMAT TabSeparated" 2>&1)
    if [ "$RESULT" != "1" ]; then
        echo "  CRITICAL: Server not responding"
        ERRORS=$((ERRORS + 1))
    else
        echo "  OK: Server is alive"
    fi
}

check_replication() {
    echo "[CHECK] Replication health..."
    BROKEN=$(run_query "
    SELECT count() FROM system.replicas
    WHERE is_readonly = 1 OR is_session_expired = 1
    FORMAT TabSeparated
    ")
    if [ "$BROKEN" -gt 0 ]; then
        echo "  WARNING: ${BROKEN} replica(s) in bad state"
        WARNINGS=$((WARNINGS + 1))
    else
        echo "  OK: All replicas healthy"
    fi
}

check_part_count() {
    echo "[CHECK] Part count..."
    HIGH_PARTS=$(run_query "
    SELECT count() FROM (
        SELECT table, count() AS cnt
        FROM system.parts
        WHERE active = 1
        GROUP BY table
        HAVING cnt > 300
    ) FORMAT TabSeparated
    ")
    if [ "$HIGH_PARTS" -gt 0 ]; then
        echo "  WARNING: ${HIGH_PARTS} table(s) with >300 parts"
        WARNINGS=$((WARNINGS + 1))
    else
        echo "  OK: Part counts are healthy"
    fi
}

check_disk_usage() {
    echo "[CHECK] Disk usage..."
    HIGH_DISK=$(run_query "
    SELECT count() FROM system.disks
    WHERE free_space / total_space < 0.15
    FORMAT TabSeparated
    ")
    if [ "$HIGH_DISK" -gt 0 ]; then
        echo "  CRITICAL: ${HIGH_DISK} disk(s) above 85% capacity"
        ERRORS=$((ERRORS + 1))
    else
        echo "  OK: Disk usage is acceptable"
    fi
}

check_long_queries() {
    echo "[CHECK] Long-running queries..."
    LONG=$(run_query "
    SELECT count() FROM system.processes
    WHERE elapsed > 300
    FORMAT TabSeparated
    ")
    if [ "$LONG" -gt 0 ]; then
        echo "  WARNING: ${LONG} query/queries running longer than 5 minutes"
        WARNINGS=$((WARNINGS + 1))
    else
        echo "  OK: No long-running queries"
    fi
}

check_liveness
check_replication
check_part_count
check_disk_usage
check_long_queries

echo ""
echo "=== Summary: ${ERRORS} error(s), ${WARNINGS} warning(s) ==="

if [ $ERRORS -gt 0 ]; then exit 2; fi
if [ $WARNINGS -gt 0 ]; then exit 1; fi
exit 0
```

## Integrating with Monitoring

Use exit codes to integrate with Nagios, Icinga, or any monitoring system that expects return code 0 (OK), 1 (WARNING), or 2 (CRITICAL).

```bash
# Kubernetes liveness probe
livenessProbe:
  exec:
    command: ["/bin/sh", "-c", "curl -sf http://localhost:8123/ping"]
  initialDelaySeconds: 30
  periodSeconds: 10
```

## Summary

A ClickHouse health check script validates liveness, replication state, part counts, disk capacity, and query runtime. Standard exit codes make it compatible with any monitoring framework, while individual check functions keep the script easy to extend or customize.
