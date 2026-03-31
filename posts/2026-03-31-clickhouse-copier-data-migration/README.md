# How to Use clickhouse-copier for Data Migration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, clickhouse-copier, Data Migration, Cluster, Resharding

Description: A practical guide to using clickhouse-copier for migrating data between ClickHouse clusters, resharding tables, and copying large datasets safely.

---

`clickhouse-copier` is a utility bundled with ClickHouse for copying data between clusters or within a cluster. It handles resharding, schema changes, and large-scale migrations without requiring downtime on the source.

## When to Use clickhouse-copier

- Migrating data from one cluster to another
- Resharding a distributed table (changing shard count or key)
- Copying a subset of data with a different schema
- Cross-datacenter replication for analytics

## Installation

`clickhouse-copier` ships with the ClickHouse server package:

```bash
which clickhouse-copier
# /usr/bin/clickhouse-copier
```

## ZooKeeper Requirement

clickhouse-copier uses ZooKeeper (or ClickHouse Keeper) to coordinate tasks and track progress:

```bash
# Ensure ZooKeeper is accessible
clickhouse-copier --zookeeper-config /etc/clickhouse-server/config.xml \
  --task-path /clickhouse/copier/task1 \
  --config copier.xml \
  --task-file task.xml \
  --base-dir /var/lib/clickhouse/copier
```

## Task Configuration File

Create `task.xml` describing source, destination, and mapping:

```xml
<clickhouse>
  <remote_servers>
    <source_cluster>
      <shard>
        <replica>
          <host>source-ch-01</host>
          <port>9000</port>
        </replica>
      </shard>
    </source_cluster>
    <destination_cluster>
      <shard>
        <replica>
          <host>dest-ch-01</host>
          <port>9000</port>
        </replica>
        <replica>
          <host>dest-ch-02</host>
          <port>9000</port>
        </replica>
      </shard>
    </destination_cluster>
  </remote_servers>

  <tables>
    <table_hits>
      <cluster_pull>source_cluster</cluster_pull>
      <database_pull>analytics</database_pull>
      <table_pull>events</table_pull>

      <cluster_push>destination_cluster</cluster_push>
      <database_push>analytics</database_push>
      <table_push>events</table_push>

      <engine>
        ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
        ORDER BY (project_id, ts)
      </engine>

      <sharding_key>sipHash64(user_id)</sharding_key>
    </table_hits>
  </tables>
</clickhouse>
```

## Running the Copy

```bash
clickhouse-copier \
  --daemon \
  --config /etc/clickhouse-server/config.xml \
  --task-path /clickhouse/copier/task1 \
  --task-file task.xml \
  --base-dir /var/lib/clickhouse/copier/task1
```

## Monitoring Progress

Track progress in ZooKeeper:

```bash
clickhouse-client --query "SELECT * FROM system.zookeeper WHERE path = '/clickhouse/copier/task1'"
```

Or check the copier log:

```bash
tail -f /var/lib/clickhouse/copier/task1/clickhouse-copier.log
```

## Handling Partial Copies

clickhouse-copier is idempotent - re-running after a failure resumes from where it stopped. Completed shards are recorded in ZooKeeper.

## Summary

`clickhouse-copier` is the standard tool for large-scale ClickHouse data migrations and resharding. Its ZooKeeper-based coordination ensures fault-tolerant, resumable copies between clusters of any size.
