# How to Build a DR Strategy for Multi-Cloud ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Disaster Recovery, Multi-Cloud, High Availability, Replication

Description: Design a disaster recovery strategy for ClickHouse spanning multiple cloud providers, covering RPO, RTO, backup policies, and failover runbooks.

---

A solid DR strategy for ClickHouse defines your Recovery Point Objective (RPO) and Recovery Time Objective (RTO) then maps each to specific ClickHouse features: replication, backups, and Keeper redundancy.

## RPO and RTO Targets

| Tier | RPO | RTO | Approach |
|---|---|---|---|
| Critical | 0 min | < 5 min | Synchronous cross-cloud replication |
| Standard | 15 min | < 30 min | Async replication + hot standby |
| Archive | 24 h | < 4 h | Daily backup to cold object storage |

## Synchronous Cross-Cloud Replication

Use `ReplicatedMergeTree` with a quorum write setting to ensure at least one cross-cloud replica has the data before acknowledging the write:

```xml
<insert_quorum>2</insert_quorum>
<insert_quorum_parallel>1</insert_quorum_parallel>
```

This guarantees RPO = 0 at the cost of slightly higher write latency.

## Scheduled Backups

Use `clickhouse-backup` to push snapshots to multi-cloud object storage:

```bash
# Back up to S3
clickhouse-backup create --config /etc/clickhouse-backup/config.yml daily_$(date +%Y%m%d)
clickhouse-backup upload --config /etc/clickhouse-backup/config.yml daily_$(date +%Y%m%d)

# Also push to GCS for cloud independence
gsutil -m cp -r backups/daily_$(date +%Y%m%d) gs://ch-dr-backups/
```

## Failover Runbook

```text
1. Detect primary region failure via monitoring alert.
2. Verify Keeper quorum is healthy in secondary region.
3. Update DNS or load balancer to point to secondary cluster.
4. Verify data consistency: SELECT max(ts) FROM events;
5. Notify stakeholders and open incident in OneUptime.
6. Begin primary region recovery procedures.
```

## Testing DR

Run quarterly DR drills:

```bash
# Simulate primary failure by blocking traffic
iptables -I INPUT -s 10.1.0.0/16 -j DROP

# Measure RTO from failover start to first successful query
time clickhouse-client --host ch-dr-node -q "SELECT count() FROM events"
```

## Monitoring

Configure [OneUptime](https://oneuptime.com) status pages for both primary and DR environments. Set up synthetic monitors that run read queries against both clusters every 30 seconds so failover is detected automatically.

## Summary

A multi-cloud ClickHouse DR strategy layers synchronous replication for RPO-zero requirements with scheduled backups for cost-effective archival recovery. Regular drills validate RTO targets before an actual incident.
