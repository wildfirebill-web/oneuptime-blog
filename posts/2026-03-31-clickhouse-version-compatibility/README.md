# How to Check ClickHouse Version Compatibility Before Upgrade

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Upgrade, Compatibility, Version, Operations

Description: Learn how to check ClickHouse version compatibility, identify breaking changes, and validate client-server version mismatch before upgrading.

---

ClickHouse version compatibility checking is a critical step before any upgrade. Breaking changes in query syntax, function behavior, or inter-server protocol can cause immediate failures after upgrading.

## Checking Current Versions

Start by knowing what version is running on each component:

```sql
-- Server version
SELECT version();

-- Replica versions in a cluster (check all nodes)
SELECT hostName(), version()
FROM clusterAllReplicas('my_cluster', system.one);
```

```bash
# Client version
clickhouse-client --version

# Check installed package version
dpkg -l clickhouse-server clickhouse-client 2>/dev/null || rpm -qa clickhouse-server clickhouse-client
```

## Understanding Compatible Version Jumps

ClickHouse supports upgrading one major version at a time within a cluster for rolling upgrades. The server-to-server protocol is backward compatible for one version:

- **Supported**: 24.3 to 24.8 (same year, incremental months)
- **Risky**: 23.x to 25.x (multi-year jump, many breaking changes)
- **Required approach for large jumps**: upgrade through intermediate LTS versions

## Checking for Breaking Changes

Query the ClickHouse release notes programmatically:

```bash
# Download and search changelog for breaking changes
curl -s https://raw.githubusercontent.com/ClickHouse/ClickHouse/master/CHANGELOG.md | \
grep -A 5 -i "breaking change\|incompatible change\|removed\|deprecated"
```

## Checking Client-Server Version Compatibility

ClickHouse clients and servers should be on the same version for full compatibility:

```sql
-- Check what version clients are connecting with
SELECT
    client_name,
    client_version_major,
    client_version_minor,
    count() AS connections
FROM system.processes
GROUP BY client_name, client_version_major, client_version_minor
ORDER BY connections DESC;
```

## Checking Table Engine Compatibility

Some upgrades change table engine behavior. Verify your engines are supported:

```sql
SELECT engine, count() AS tables
FROM system.tables
WHERE database NOT IN ('system', 'information_schema', '_temporary_and_external_tables')
GROUP BY engine
ORDER BY tables DESC;
```

Check the changelog for any changes to engines you use, especially `MergeTree`, `ReplacingMergeTree`, `CollapsingMergeTree`, and `AggregatingMergeTree`.

## Checking Deprecated Functions

Identify queries using deprecated functions that may be removed:

```sql
SELECT
    query,
    event_time
FROM system.query_log
WHERE
    type = 'QueryFinish'
    AND event_time >= now() - INTERVAL 7 DAY
    AND (
        query LIKE '%arrayEnumerateDense%'
        OR query LIKE '%arrayEnumerateUniq%'
        OR query LIKE '%yandexConsistentHash%'
    )
LIMIT 20;
```

Adjust the list based on the specific deprecations in your target version.

## Validating ZooKeeper/Keeper Compatibility

If upgrading ClickHouse Keeper or ZooKeeper alongside ClickHouse:

```sql
-- Check Keeper version compatibility
SELECT * FROM system.zookeeper WHERE path = '/';
```

Check the official compatibility matrix for Keeper versions.

## Pre-Upgrade Compatibility Test Script

```bash
#!/bin/bash
TARGET_VERSION="24.8"
CURRENT_VERSION=$(clickhouse-client --query "SELECT version()" 2>/dev/null)

echo "Current version: $CURRENT_VERSION"
echo "Target version: $TARGET_VERSION"

# Check if target package exists
apt-cache show clickhouse-server="${TARGET_VERSION}.*" 2>/dev/null && \
    echo "Package available" || echo "Package NOT found in repository"

# List any known breaking changes
echo "Review changelog: https://github.com/ClickHouse/ClickHouse/blob/master/CHANGELOG.md"
```

## Summary

ClickHouse version compatibility checking involves verifying server and client versions on all nodes, reviewing changelogs for breaking changes relevant to your table engines and query patterns, and checking for deprecated function usage in recent query history. Upgrade through intermediate versions for large version jumps and always test in staging before production.
