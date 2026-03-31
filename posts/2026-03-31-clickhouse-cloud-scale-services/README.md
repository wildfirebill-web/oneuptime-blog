# How to Scale ClickHouse Cloud Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ClickHouse Cloud, Scaling, Vertical Scaling, Horizontal Scaling, Performance

Description: Learn how to scale ClickHouse Cloud services vertically by adjusting memory and compute, and understand the scaling model for production workloads.

---

ClickHouse Cloud uses a shared-nothing architecture with compute and storage separated. Scaling is primarily vertical - you adjust the total memory allocated to the service, which in turn determines CPU and concurrency capacity.

## Understanding the Scaling Model

ClickHouse Cloud services are defined by total memory (in GB). Memory directly controls:
- Number of replicas (each replica gets a share of total memory)
- CPU cores per replica
- Query concurrency capacity

For Production tier, the minimum is 24 GB total memory (2 replicas x 12 GB each).

## Scaling via the Console

1. Open your service in the ClickHouse Cloud console
2. Go to "Settings" - "Compute"
3. Adjust the memory slider
4. Click "Save changes"

The change is applied with no downtime on Production tier.

## Scaling via the API

```bash
curl -X PATCH https://api.clickhouse.cloud/v1/organizations/{orgId}/services/{serviceId} \
  -H "Authorization: Bearer $CLICKHOUSE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "minTotalMemoryGb": 48,
    "maxTotalMemoryGb": 192
  }'
```

Setting `minTotalMemoryGb` and `maxTotalMemoryGb` to the same value disables auto-scaling and pins the service at a fixed size.

## Checking Current Service Size

```bash
curl https://api.clickhouse.cloud/v1/organizations/{orgId}/services/{serviceId} \
  -H "Authorization: Bearer $CLICKHOUSE_API_KEY" \
  | jq '{minMemory: .service.minTotalMemoryGb, maxMemory: .service.maxTotalMemoryGb}'
```

## When to Scale Up

Monitor these signals to know when to increase capacity:

```sql
-- Check for memory-limited queries
SELECT count()
FROM system.query_log
WHERE event_time > now() - INTERVAL 1 HOUR
  AND exception LIKE '%Memory limit%';

-- Check average query concurrency
SELECT
    toStartOfMinute(event_time) AS minute,
    count() AS queries
FROM system.query_log
WHERE event_time > now() - INTERVAL 1 HOUR
  AND type = 'QueryStart'
GROUP BY minute
ORDER BY minute;
```

## Horizontal Scaling with Read Replicas

For read-heavy workloads, ClickHouse Cloud supports adding read replicas in the same or different regions. This is configured through the console under "Replicas".

## Summary

Scaling ClickHouse Cloud means adjusting total memory allocation, which proportionally increases CPU and concurrency. Use the API or console for on-demand scaling, set different min/max values to enable auto-scaling, and monitor query log for memory errors as signals to scale up.
