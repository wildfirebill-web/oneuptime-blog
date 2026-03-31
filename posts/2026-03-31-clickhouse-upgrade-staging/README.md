# How to Test ClickHouse Upgrades in a Staging Environment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Upgrade, Staging, Testing, Operations

Description: Learn how to set up a ClickHouse staging environment, replicate production workloads, and validate upgrades before rolling out to production.

---

Testing ClickHouse upgrades in a staging environment prevents production outages caused by breaking changes, configuration incompatibilities, or unexpected behavior differences between versions.

## Setting Up a Staging Environment

Your staging environment should mirror production as closely as possible:

```bash
# Use Docker Compose for a local staging cluster
cat > docker-compose.yml << 'EOF'
version: '3'
services:
  clickhouse-staging:
    image: clickhouse/clickhouse-server:24.8
    ports:
      - "18123:8123"
      - "19000:9000"
    volumes:
      - ./config:/etc/clickhouse-server/config.d
      - ./staging-data:/var/lib/clickhouse
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
EOF
docker-compose up -d
```

## Restoring Production Schema to Staging

Copy table schemas (not data) from production:

```bash
# Export all CREATE TABLE statements
clickhouse-client --host production-host --query "
SELECT create_table_query
FROM system.tables
WHERE database NOT IN ('system', 'information_schema', '_temporary_and_external_tables')
FORMAT TSVRaw" > /tmp/production_schema.sql

# Import to staging
clickhouse-client --host localhost --port 19000 < /tmp/production_schema.sql
```

## Loading a Sample of Production Data

Load a representative sample for testing:

```sql
-- On production: export sample
INSERT INTO FUNCTION remoteSecure('staging-host:9000', 'default', 'events_sample', 'default', '')
SELECT * FROM events
WHERE event_time >= today() - 7
LIMIT 10000000;
```

## Upgrading the Staging Instance

```bash
# Stop staging ClickHouse
docker stop clickhouse-staging

# Pull new version
docker pull clickhouse/clickhouse-server:25.1

# Update docker-compose.yml image version
sed -i 's/clickhouse-server:24.8/clickhouse-server:25.1/' docker-compose.yml

# Start with new version
docker-compose up -d
```

## Running Validation Queries

Execute your critical production queries against staging after upgrade:

```sql
-- Test key analytical queries
EXPLAIN SELECT
    toDate(event_time) AS day,
    count() AS events,
    uniq(user_id) AS users
FROM events_sample
WHERE event_time >= today() - 30
GROUP BY day
ORDER BY day;
```

Create a test suite of your most important queries:

```bash
#!/bin/bash
STAGING="clickhouse-client --host localhost --port 19000"
PASS=0
FAIL=0

run_test() {
    local name="$1"
    local query="$2"
    if $STAGING --query "$query" > /dev/null 2>&1; then
        echo "PASS: $name"
        PASS=$((PASS + 1))
    else
        echo "FAIL: $name"
        FAIL=$((FAIL + 1))
    fi
}

run_test "Basic count" "SELECT count() FROM events_sample"
run_test "Date aggregation" "SELECT toDate(event_time), count() FROM events_sample GROUP BY 1 LIMIT 1"
run_test "User funnel" "SELECT user_id, count() FROM events_sample GROUP BY user_id ORDER BY count() DESC LIMIT 10"

echo "Results: $PASS passed, $FAIL failed"
```

## Comparing Results Between Versions

Run the same queries on both staging (new version) and production (old version) and compare results:

```bash
PROD_RESULT=$(clickhouse-client --host production --query "SELECT count() FROM events WHERE event_time >= today() - 7")
STAGING_RESULT=$(clickhouse-client --host staging --port 19000 --query "SELECT count() FROM events_sample")

echo "Production: $PROD_RESULT"
echo "Staging:    $STAGING_RESULT"
```

## Checking for Performance Regressions

Compare query execution times between versions:

```sql
-- On staging: check execution time
SELECT
    query,
    query_duration_ms,
    read_rows
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_time >= now() - INTERVAL 1 HOUR
ORDER BY query_duration_ms DESC
LIMIT 20;
```

## Summary

Testing ClickHouse upgrades in staging requires an environment that mirrors production configuration, a sample of production data, and a test suite of critical queries. Run all key queries, check for performance regressions, and validate results match expected values before scheduling a production upgrade. Document any differences or issues found during staging testing.
