# How to Use system.clusters Table in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cluster, Distributed, System Table, Administration

Description: Learn how to query system.clusters to inspect cluster topology, shard layout, replica addresses, and connection parameters across your ClickHouse deployment.

---

The `system.clusters` table is the primary source of truth for your ClickHouse cluster topology. It exposes every cluster defined in your server configuration, showing each shard, its replicas, hostnames, ports, and whether inter-shard compression is enabled. You query it like any other table, making it easy to audit cluster shape, verify configuration drift, and drive dynamic Distributed table creation.

## What system.clusters Contains

```sql
DESCRIBE system.clusters;
```

Key columns:

| Column | Type | Meaning |
|---|---|---|
| `cluster` | String | Cluster name from config |
| `shard_num` | UInt32 | Shard index (1-based) |
| `shard_weight` | UInt32 | Relative weight for data distribution |
| `replica_num` | UInt32 | Replica index within the shard |
| `host_name` | String | Hostname of this replica |
| `host_address` | String | Resolved IP address |
| `port` | UInt16 | TCP port |
| `user` | String | Username used for inter-shard queries |
| `is_local` | UInt8 | 1 if this row represents the local node |
| `errors_count` | UInt32 | Cumulative connection errors |
| `slowdowns_count` | UInt32 | Cumulative hedged request slowdowns |
| `estimated_recovery_time` | UInt32 | Seconds until the replica is expected to recover |

## Basic Query: List All Clusters

```sql
SELECT
    cluster,
    shard_num,
    replica_num,
    host_name,
    port,
    is_local
FROM system.clusters
ORDER BY cluster, shard_num, replica_num;
```

This returns every node in every configured cluster. If you have a single-node setup, you will see the built-in `default` cluster plus any additional clusters you defined.

## Count Shards and Replicas Per Cluster

```sql
SELECT
    cluster,
    countDistinct(shard_num)   AS shards,
    count()                    AS total_replicas,
    count() / countDistinct(shard_num) AS replicas_per_shard
FROM system.clusters
GROUP BY cluster
ORDER BY cluster;
```

This gives a quick summary of cluster shape. A properly configured three-shard, two-replica cluster will show `shards = 3`, `total_replicas = 6`, `replicas_per_shard = 2`.

## Find the Local Node

```sql
SELECT
    cluster,
    shard_num,
    replica_num,
    host_name,
    port
FROM system.clusters
WHERE is_local = 1;
```

Useful in scripts that need to identify which shard a node belongs to without relying on external configuration files.

## Inspect Connection Errors

```sql
SELECT
    cluster,
    host_name,
    port,
    errors_count,
    slowdowns_count,
    estimated_recovery_time
FROM system.clusters
WHERE errors_count > 0
ORDER BY errors_count DESC;
```

Nodes with a non-zero `errors_count` have had recent inter-shard TCP failures. `estimated_recovery_time > 0` means ClickHouse considers the node temporarily unhealthy and is routing around it.

## Verify Shard Weights

```sql
SELECT
    cluster,
    shard_num,
    shard_weight,
    host_name
FROM system.clusters
WHERE cluster = 'my_cluster'
ORDER BY shard_num;
```

The `shard_weight` column controls how much data the Distributed engine writes to each shard. Equal weights mean uniform distribution. Unequal weights bias writes toward higher-weight shards, which is useful when shards have different storage capacity.

## Generate a CREATE TABLE Statement Dynamically

You can use `system.clusters` to confirm the cluster name before executing DDL:

```sql
-- Verify the cluster name exists before running DDL
SELECT cluster
FROM system.clusters
WHERE cluster = 'my_cluster'
LIMIT 1;
```

```sql
-- Then create the distributed table referencing that cluster
CREATE TABLE hits_distributed ON CLUSTER my_cluster
(
    event_date Date,
    event_time DateTime,
    user_id    UInt64,
    url        String
)
ENGINE = Distributed(my_cluster, default, hits_local, rand());
```

## Check Whether a Specific Host Is in Any Cluster

```sql
SELECT
    cluster,
    shard_num,
    replica_num,
    host_name,
    port
FROM system.clusters
WHERE host_name LIKE '%clickhouse-03%'
   OR host_address = '10.0.0.3';
```

## Export Cluster Topology as JSON

```sql
SELECT
    cluster,
    shard_num,
    replica_num,
    host_name,
    port,
    is_local,
    errors_count
FROM system.clusters
FORMAT JSONEachRow;
```

Pipe this output into a monitoring script or topology dashboard.

## Use system.clusters in a Shell Script

```bash
#!/usr/bin/env bash
# Print the number of shards in each cluster

clickhouse-client \
  --host 127.0.0.1 \
  --port 9000 \
  --query "
    SELECT
        cluster,
        countDistinct(shard_num) AS shards,
        count() AS replicas
    FROM system.clusters
    GROUP BY cluster
    ORDER BY cluster
    FORMAT TSV
  "
```

## Practical Cluster Health Check

Combine `system.clusters` with a remote connection test using the `clusterAllReplicas()` table function:

```sql
-- Query a lightweight system table on every replica at once
SELECT
    hostName()  AS replica_host,
    uptime()    AS uptime_seconds,
    version()   AS ch_version
FROM clusterAllReplicas('my_cluster', system.one)
ORDER BY replica_host;
```

This leverages the cluster definition from `system.clusters` internally and is the fastest way to confirm all replicas are alive and running the same version.

## Common Pitfalls

- `system.clusters` reflects the configuration file on the local node only. If nodes have divergent configs, each node will show different data. Always query the same node for a consistent view.
- The `errors_count` and `slowdowns_count` columns are cumulative since the last server start. A node reboot resets them to zero. Track rates rather than absolute values in alerting.
- `is_local = 1` rows can appear in multiple clusters if the same node is referenced by more than one cluster definition. This is expected in multi-cluster setups.

## Summary

`system.clusters` is the starting point for any cluster administration task in ClickHouse. Query it to audit topology, detect unhealthy replicas, validate shard weights, and generate dynamic DDL. Combine it with `clusterAllReplicas()` and the `Distributed` engine to build reliable multi-shard pipelines.
