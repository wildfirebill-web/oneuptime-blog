# How to Handle ClickHouse Version Mismatch in Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Version, Upgrade, Cluster, Operation

Description: Detect and resolve ClickHouse version mismatches in clusters that cause replication failures, feature incompatibilities, and inconsistent query behavior.

---

Running different ClickHouse versions across cluster nodes causes replication protocol mismatches and can prevent parts from being exchanged between replicas. Version mismatches must be resolved quickly.

## Detect Version Mismatch

```sql
SELECT
    host_name,
    host_address,
    version()
FROM clusterAllReplicas('prod', system.one)
ORDER BY host_name;
```

Or from the shell:

```bash
for HOST in ch-01 ch-02 ch-03; do
    echo -n "$HOST: "
    clickhouse-client --host $HOST -q "SELECT version()"
done
```

## Understanding Compatibility

ClickHouse maintains compatibility within the same minor version series. Mixing patch versions (e.g., 24.3.1 and 24.3.5) is generally safe. Mixing minor versions (24.3 and 24.4) may cause issues with new data format features.

The `interserver_http_port` (9009) is used for replication. If a node on 24.4 generates a part format unrecognized by a 24.3 node, replication errors appear:

```sql
SELECT last_exception
FROM system.replication_queue
WHERE last_exception != ''
LIMIT 5;
```

## Rolling Upgrade (Safe Order)

1. Upgrade Keeper nodes first (they are backward compatible).
2. Upgrade replica nodes one at a time.
3. After each upgrade, verify replica health before proceeding.

```bash
# Upgrade one node
apt-get install clickhouse-server=24.4.1.1

# Restart
systemctl restart clickhouse-server

# Verify health before next node
clickhouse-client -q "SELECT is_readonly, absolute_delay FROM system.replicas WHERE table = 'events'"
```

## Emergency Downgrade

If upgrading caused replication failure, downgrade the affected node:

```bash
apt-get install clickhouse-server=24.3.5.1
systemctl restart clickhouse-server
```

Wait for replication to stabilize, then re-attempt the upgrade.

## Preventing Future Mismatches

Use a configuration management tool to ensure all nodes run the same version:

```yaml
# Ansible variable
clickhouse_version: "24.4.1.1"

# Task
- name: Install ClickHouse
  apt:
    name: "clickhouse-server={{ clickhouse_version }}"
    state: present
```

Alert via [OneUptime](https://oneuptime.com) when the daily version check query finds heterogeneous versions across the cluster.

## Summary

Version mismatches in ClickHouse clusters are detected with a `clusterAllReplicas` query and resolved through a careful rolling upgrade. Always upgrade one node at a time and verify replication health before proceeding to the next.
