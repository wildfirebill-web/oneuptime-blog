# How to Configure ClickHouse Cloud for Multi-Region Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ClickHouse Cloud, Multi-Region, High Availability, Disaster Recovery, Replication

Description: Learn how to architect multi-region ClickHouse Cloud deployments for disaster recovery, low-latency reads, and compliance with data residency requirements.

---

ClickHouse Cloud services are single-region by default, but you can architect multi-region deployments for disaster recovery, geographic distribution of reads, or data residency compliance. This guide covers the patterns for achieving multi-region ClickHouse Cloud setups.

## Why Multi-Region?

- **Disaster recovery**: Service continues if one region fails
- **Low-latency reads**: Users in different regions query the nearest service
- **Data residency**: Keep EU data in EU regions, US data in US regions
- **Compliance**: Regulatory requirements for data locality

## Pattern 1 - Active-Passive with S3 Replication

Run a primary service in one region and replicate data to a standby service via S3:

```bash
# On primary: export new data hourly
clickhouse client --host primary.clickhouse.cloud --secure \
  --query "INSERT INTO FUNCTION s3(
    's3://replication-bucket/events/{_partition_id}.parquet',
    'KEY', 'SECRET', 'Parquet'
  ) SELECT * FROM events WHERE event_date = today()"
```

```bash
# On standby: import from S3
clickhouse client --host standby.clickhouse.cloud --secure \
  --query "INSERT INTO events
  SELECT * FROM s3(
    's3://replication-bucket/events/*.parquet',
    'KEY', 'SECRET', 'Parquet'
  )"
```

## Pattern 2 - Dual-Write from Application

Write data to both regions simultaneously from your application:

```python
# Pseudo-code
for region in ["us-east-1", "eu-west-1"]:
    client[region].execute("INSERT INTO events VALUES", rows)
```

This ensures real-time consistency but doubles write load and cost.

## Pattern 3 - Kafka Fan-Out

Use Kafka topics with consumers in each region:

```text
Producer -> Kafka Topic -> Consumer (US service)
                        -> Consumer (EU service)
```

Both services consume the same topic, staying in sync with a small lag.

## Setting Up Read Replica in Another Region

ClickHouse Cloud supports cross-region read replicas for Production services. Configure in the console under "Replicas" - "Add Replica" and select the target region.

## Routing Traffic by Region

Use your load balancer or application config to route users to the nearest service:

```text
US users -> us-east-1.service.clickhouse.cloud
EU users -> eu-west-1.service.clickhouse.cloud
```

## Monitoring Replication Lag

```sql
-- On standby: check latest data timestamp
SELECT max(event_date) AS latest_data FROM events;

-- Compare with primary to calculate lag
```

## Summary

Multi-region ClickHouse Cloud deployments use S3-based replication, dual-write patterns, or Kafka fan-out to keep services in sync across regions. Use ClickHouse Cloud's native cross-region read replica feature when available, and route application traffic to the nearest service for optimal latency.
