# How to Fix "Unable to resolve host" for ClickHouse Distributed Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, DNS, Distributed, Error, Troubleshooting

Description: Fix "Unable to resolve host" errors in ClickHouse distributed queries by correcting hostname configuration, DNS setup, and /etc/hosts entries.

---

"Unable to resolve host" errors in ClickHouse distributed queries mean that DNS resolution failed for one or more shard hostnames configured in the cluster definition. ClickHouse cannot connect to a shard it cannot resolve.

## Identify the Failing Hostname

The error message includes the hostname:

```text
Code: 198. DB::Exception: Unable to resolve host (clickhouse-shard3.internal):
  Name or service not known (version 24.3.3).
```

## Check DNS Resolution on the Server

```bash
# Test resolution
dig clickhouse-shard3.internal
nslookup clickhouse-shard3.internal
host clickhouse-shard3.internal

# Check /etc/resolv.conf
cat /etc/resolv.conf
```

## Use /etc/hosts as a Fallback

For small clusters, add entries to `/etc/hosts` on every ClickHouse node:

```bash
sudo tee -a /etc/hosts << 'EOF'
10.0.1.10  clickhouse-shard1.internal clickhouse-shard1
10.0.1.11  clickhouse-shard2.internal clickhouse-shard2
10.0.1.12  clickhouse-shard3.internal clickhouse-shard3
EOF
```

## Switch to IP Addresses in Cluster Config

For environments where DNS is unreliable, use IP addresses in `config.xml`:

```xml
<remote_servers>
  <my_cluster>
    <shard>
      <replica>
        <host>10.0.1.10</host>
        <port>9000</port>
      </replica>
    </shard>
    <shard>
      <replica>
        <host>10.0.1.11</host>
        <port>9000</port>
      </replica>
    </shard>
    <shard>
      <replica>
        <host>10.0.1.12</host>
        <port>9000</port>
      </replica>
    </shard>
  </my_cluster>
</remote_servers>
```

## Configure DNS Caching in ClickHouse

ClickHouse caches DNS results. If a hostname recently changed IP:

```sql
-- Force DNS cache refresh
SYSTEM DROP DNS CACHE;
```

Configure the cache TTL:

```xml
<dns_cache_update_period>15</dns_cache_update_period>
```

## Verify Cluster Sees All Shards

After fixing DNS:

```sql
SELECT cluster, shard_num, replica_num, host_name, errors_count
FROM system.clusters
WHERE cluster = 'my_cluster';
```

High `errors_count` indicates persistent resolution failures.

## Use Kubernetes Service DNS

In Kubernetes environments, use the fully qualified service DNS names:

```xml
<host>clickhouse-shard1.clickhouse-namespace.svc.cluster.local</host>
```

## Summary

"Unable to resolve host" in ClickHouse distributed queries is a DNS problem. Fix it by adding `/etc/hosts` entries for all shards, switching to IP addresses in cluster config for reliability, running `SYSTEM DROP DNS CACHE` after hostname changes, and using fully qualified Kubernetes service DNS names in container deployments.
