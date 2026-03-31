# How to Fix "All connection tries failed" in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Connection, Distributed, Error, Troubleshooting

Description: Resolve "All connection tries failed" errors in ClickHouse distributed queries by fixing shard connectivity, authentication, and cluster config.

---

"All connection tries failed" in ClickHouse appears when a distributed query cannot connect to any replica of a shard. ClickHouse tries all configured replicas and gives up when all fail. This error is common after cluster configuration changes, network disruptions, or shard restarts.

## Identify the Failing Shard

The error message usually includes the shard address:

```text
Code: 519. DB::Exception: All connection tries failed.
Shard 1 failed: clickhouse-shard1:9000 - Connection refused
```

## Test Connectivity to Each Shard

From the coordinator node:

```bash
# Test native protocol
nc -zv clickhouse-shard1 9000
nc -zv clickhouse-shard2 9000

# Test HTTP interface
curl -s "http://clickhouse-shard1:8123/ping"
```

## Verify Cluster Configuration

```sql
SELECT cluster, shard_num, replica_num, host_name, port, is_local
FROM system.clusters
WHERE cluster = 'my_cluster';
```

Ensure hostnames in `config.xml` are resolvable:

```bash
dig +short clickhouse-shard1
# Should return an IP address
```

## Fix DNS Resolution

If hostnames are not resolving, use IP addresses directly in cluster config, or fix DNS:

```xml
<remote_servers>
  <my_cluster>
    <shard>
      <replica>
        <host>192.168.1.10</host>
        <port>9000</port>
      </replica>
    </shard>
  </my_cluster>
</remote_servers>
```

## Fix Authentication Failures

If the shard is running but authentication fails:

```xml
<remote_servers>
  <my_cluster>
    <shard>
      <replica>
        <host>clickhouse-shard1</host>
        <port>9000</port>
        <user>interserver_user</user>
        <password>secret</password>
      </replica>
    </shard>
  </my_cluster>
</remote_servers>
```

## Configure Failover Behavior

For read queries, configure what happens when all replicas of a shard fail:

```sql
-- Skip unavailable shards (returns partial results)
SET skip_unavailable_shards = 1;

SELECT count() FROM distributed_table;
```

## Check Inter-Server HTTP Port

Replication and cluster communication also uses the inter-server HTTP port (9009 by default):

```bash
nc -zv clickhouse-shard1 9009
```

Ensure this port is allowed in firewall rules.

## Summary

"All connection tries failed" is a connectivity error in ClickHouse distributed queries. Verify shard hostnames resolve correctly, test port connectivity from the coordinator, check authentication credentials in the cluster config, and use `skip_unavailable_shards = 1` to allow partial results when some shards are temporarily unavailable.
