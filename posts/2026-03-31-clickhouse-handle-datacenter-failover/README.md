# How to Handle ClickHouse Data Center Failover

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Failover, Data Center, High Availability, Disaster Recovery

Description: Execute a ClickHouse data center failover with minimal data loss by following a structured runbook covering Keeper quorum, replica promotion, and traffic cutover.

---

Data center failover is the highest-severity ClickHouse incident. A well-practiced runbook reduces RTO from hours to minutes.

## Prerequisites

- ClickHouse nodes in at least two data centers.
- Keeper ensemble with a quorum node in the DR data center.
- DNS or load balancer that can be updated quickly.

## Phase 1 - Assess the Situation (0-5 min)

Determine whether the primary DC is completely down or partially reachable:

```bash
# Test connectivity to primary DC nodes
for HOST in ch-dc1-01 ch-dc1-02; do
    ping -c 3 $HOST && echo "UP" || echo "DOWN"
done

# Check Keeper quorum status from DR DC
clickhouse-keeper-client --host keeper-dc2 -q "stat" 2>&1 | grep -i "mode\|leader"
```

## Phase 2 - Verify DR Replica Health (5-10 min)

```sql
-- Run from a DR DC node
SELECT
    table,
    is_leader,
    absolute_delay,
    queue_size
FROM system.replicas
WHERE absolute_delay < 60  -- replicas that are current
ORDER BY table;
```

If `absolute_delay > 60` on all DR replicas, you may lose up to that many seconds of data.

## Phase 3 - Promote DR Cluster (10-15 min)

If using DNS:

```bash
# Update DNS TTL ahead of time (set to 60s weeks in advance)
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "clickhouse.internal",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [{"Value": "ch-dc2-lb.internal"}]
      }
    }]
  }'
```

## Phase 4 - Redirect Application Traffic

Update application `CLICKHOUSE_HOST` environment variables or connection pool config to point to DR nodes. Restart affected services.

## Phase 5 - Verify and Monitor

```sql
SELECT count() FROM events WHERE ts >= now() - INTERVAL 5 MINUTE;
```

Monitor query throughput and error rates via [OneUptime](https://oneuptime.com) status page. Communicate status updates to stakeholders every 15 minutes.

## Phase 6 - Return to Primary DC

After primary DC recovery, perform a replication catch-up before redirecting traffic back:

```sql
SELECT absolute_delay FROM system.replicas WHERE table = 'events';
-- Wait until delay = 0, then cut back over
```

## Summary

Data center failover succeeds when Keeper quorum survives the outage, DR replicas are current, and DNS or load balancer updates are pre-tested. Practice the runbook quarterly to keep RTO under 15 minutes.
