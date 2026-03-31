# How to Implement Automatic Failover for ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Failover, High Availability, Replication, Cluster

Description: Learn how to implement automatic failover for ClickHouse so that queries continue to succeed when a node goes down without manual intervention.

---

## How ClickHouse Handles Failover

ClickHouse does not have a built-in automatic failover agent like MySQL's MHA. Instead, failover relies on a combination of `ReplicatedMergeTree`, `Distributed` tables, and an external proxy or load balancer to detect failures and redirect traffic.

## Replica-Level Failover via Distributed Tables

When you use a `Distributed` table with multiple replicas per shard, ClickHouse automatically skips unresponsive replicas and retries against available ones. Set `load_balancing` to control how replicas are chosen:

```sql
SET load_balancing = 'in_order';
-- Tries replicas in order; falls back to the next on failure

SET load_balancing = 'nearest_hostname';
-- Prefers the replica with the closest hostname
```

## Configuring the Cluster with Replica Priority

```xml
<remote_servers>
  <prod_cluster>
    <shard>
      <internal_replication>true</internal_replication>
      <replica>
        <host>ch-primary</host>
        <port>9000</port>
        <priority>1</priority>
      </replica>
      <replica>
        <host>ch-standby</host>
        <port>9000</port>
        <priority>2</priority>
      </replica>
    </shard>
  </prod_cluster>
</remote_servers>
```

Lower priority values are preferred. The standby is only used when the primary is unavailable.

## Using chproxy for Automatic Failover

`chproxy` is a popular HTTP proxy for ClickHouse that supports automatic failover:

```yaml
clusters:
  - name: prod
    nodes:
      - ch-primary:8123
      - ch-standby:8123
    heartbeat_interval: 5s
    death_count: 3
    death_duration: 30s
```

When `ch-primary` fails three consecutive health checks within 30 seconds, `chproxy` routes traffic to `ch-standby` automatically.

## Configuring HAProxy for TCP Failover

```text
backend clickhouse_backend
    balance roundrobin
    option tcp-check
    server ch-primary 10.0.0.1:9000 check inter 5s fall 3 rise 2
    server ch-standby 10.0.0.2:9000 check inter 5s fall 3 rise 2 backup
```

The `backup` flag means `ch-standby` only receives traffic when all non-backup servers are down.

## Monitoring Failover Readiness

Check that your standby replica is in sync before relying on it:

```sql
SELECT
    replica_name,
    is_readonly,
    queue_size,
    log_max_index - log_pointer AS lag_entries
FROM system.replicas
WHERE table = 'events_local';
```

A `lag_entries` value of 0 and `is_readonly = 0` means the replica is ready.

## Testing Failover

```bash
# Simulate primary failure
docker stop ch-primary

# Verify queries still work through the Distributed table
clickhouse-client --host ch-lb --query "SELECT count() FROM events_distributed"
```

## Summary

Automatic ClickHouse failover combines `ReplicatedMergeTree` for data synchronization with a proxy layer (chproxy or HAProxy) that detects node failures and reroutes traffic. Distributed tables handle read-level failover natively. Always test failover in a staging environment and monitor `system.replicas` to ensure replicas are synchronized before depending on them.
