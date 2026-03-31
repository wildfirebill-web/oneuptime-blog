# ClickHouse Production Readiness Checklist

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Production, Checklist, Operations, Deployment

Description: A comprehensive checklist for deploying ClickHouse to production, covering hardware, security, replication, monitoring, and backup requirements.

---

Deploying ClickHouse to production requires careful preparation across infrastructure, security, data integrity, and observability. This checklist ensures you have covered the critical bases before going live.

## Hardware and Infrastructure

```text
[ ] Dedicated NVMe SSD for ClickHouse data directory (/var/lib/clickhouse)
[ ] Separate disk for ClickHouse logs to avoid competing I/O
[ ] Minimum 32GB RAM for analytics workloads (128GB+ for heavy aggregations)
[ ] At least 8 CPU cores; 32+ for high concurrency
[ ] 10Gbps network between ClickHouse nodes
[ ] Filesystem: ext4 or xfs with noatime mount option
```

## Replication and High Availability

```sql
-- Verify replication is configured
SELECT table, is_leader, total_replicas, active_replicas
FROM system.replicas
WHERE is_readonly = 0;

-- Check for replication lag
SELECT table, absolute_delay
FROM system.replicas
WHERE absolute_delay > 300;  -- Alert if >5 minutes behind
```

```text
[ ] At least 2 replicas per shard
[ ] ClickHouse Keeper with 3 nodes (or external ZooKeeper cluster)
[ ] Test replica failover before go-live
[ ] Distributed table created on top of replicated tables
```

## Security

```text
[ ] Default user password set (not empty)
[ ] Network access restricted via /etc/clickhouse-server/config.d/network.xml
[ ] TLS enabled for client connections (port 8443/9440)
[ ] TLS enabled for inter-node replication
[ ] Row-level security configured for multi-tenant deployments
[ ] Audit logging enabled
```

```xml
<listen_host>0.0.0.0</listen_host>
<https_port>8443</https_port>
<tcp_port_secure>9440</tcp_port_secure>
```

## Schema and Data Integrity

```sql
-- Verify table schemas are correct
DESCRIBE TABLE your_main_table;

-- Check for any failed mutations
SELECT database, table, command, is_done, latest_fail_reason
FROM system.mutations
WHERE is_done = 0;
```

```text
[ ] Partitioning strategy matches data retention requirements
[ ] TTL rules defined for data expiry
[ ] ORDER BY columns chosen for primary query patterns
[ ] LowCardinality applied to string columns with < 10,000 distinct values
```

## Monitoring

```text
[ ] Metrics exported to Prometheus via /metrics endpoint
[ ] Alerts for: disk usage >80%, replication lag >5min, merge queue depth >100
[ ] system.query_log retention configured
[ ] system.metric_log retention configured
[ ] Grafana dashboard for ClickHouse internals deployed
```

## Backup

```text
[ ] clickhouse-backup or BACKUP TO S3 configured
[ ] Backup schedule defined and tested
[ ] Restore procedure documented and tested
[ ] Backup retention policy configured
```

## Load Testing

```text
[ ] clickhouse-benchmark run against production-size dataset
[ ] Peak concurrent query load tested
[ ] Memory limits set appropriately (max_memory_usage, max_server_memory_usage)
[ ] Query timeout set (max_execution_time)
```

## Summary

Production ClickHouse deployments succeed when infrastructure, replication, security, monitoring, and backup are all in place before the first production traffic arrives. Walk through each section of this checklist and document the results to ensure nothing is overlooked in the rush to go live.
