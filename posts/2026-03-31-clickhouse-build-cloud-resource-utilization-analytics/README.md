# How to Build Cloud Resource Utilization Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cloud Resource, Utilization, FinOps, Right-Sizing

Description: Learn how to analyze cloud resource utilization in ClickHouse to identify over-provisioned instances, idle resources, and right-sizing opportunities.

---

Cloud over-provisioning is expensive. By analyzing CPU, memory, and network utilization data in ClickHouse, platform and FinOps teams can identify idle resources, right-size instances, and reduce waste without impacting performance.

## Schema

```sql
CREATE TABLE cloud_resource_metrics
(
    ts              DateTime,
    provider        LowCardinality(String),
    resource_id     String,
    resource_type   LowCardinality(String), -- 'ec2','rds','elasticache','gce'
    region          LowCardinality(String),
    team            LowCardinality(String),
    instance_type   LowCardinality(String),
    cpu_pct         Float32,
    mem_pct         Float32,
    net_in_mbps     Float32,
    net_out_mbps    Float32,
    hourly_cost_usd Decimal(8, 4)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (resource_id, ts);
```

## Average and Peak CPU by Instance

```sql
SELECT
    resource_id,
    instance_type,
    team,
    avg(cpu_pct)  AS avg_cpu,
    max(cpu_pct)  AS peak_cpu,
    sum(hourly_cost_usd) AS total_cost_usd
FROM cloud_resource_metrics
WHERE ts >= now() - INTERVAL 7 DAY
GROUP BY resource_id, instance_type, team
ORDER BY avg_cpu ASC
LIMIT 50;
```

## Idle Resources (CPU below 5% all week)

```sql
SELECT
    resource_id,
    instance_type,
    team,
    avg(cpu_pct) AS avg_cpu,
    sum(hourly_cost_usd) AS wasted_cost
FROM cloud_resource_metrics
WHERE ts >= now() - INTERVAL 7 DAY
GROUP BY resource_id, instance_type, team
HAVING avg_cpu < 5 AND count() > 100
ORDER BY wasted_cost DESC;
```

## Right-Sizing Candidates

Find instances where peak CPU never exceeds 30%:

```sql
SELECT
    resource_id,
    instance_type,
    avg(cpu_pct)     AS avg_cpu,
    max(cpu_pct)     AS peak_cpu,
    avg(mem_pct)     AS avg_mem,
    sum(hourly_cost_usd) AS monthly_cost
FROM cloud_resource_metrics
WHERE ts >= now() - INTERVAL 30 DAY
GROUP BY resource_id, instance_type
HAVING peak_cpu < 30 AND avg_mem < 40
ORDER BY monthly_cost DESC
LIMIT 20;
```

## Utilization Heatmap by Hour of Day

```sql
SELECT
    toHour(ts)     AS hour_of_day,
    resource_type,
    avg(cpu_pct)   AS avg_cpu
FROM cloud_resource_metrics
WHERE ts >= now() - INTERVAL 30 DAY
GROUP BY hour_of_day, resource_type
ORDER BY hour_of_day, resource_type;
```

## Cost Per CPU Unit

Efficiency metric - lower is better:

```sql
SELECT
    resource_id,
    instance_type,
    sum(hourly_cost_usd) / (avg(cpu_pct) / 100.0 + 0.001) AS cost_per_cpu_unit
FROM cloud_resource_metrics
WHERE ts >= now() - INTERVAL 7 DAY
GROUP BY resource_id, instance_type
ORDER BY cost_per_cpu_unit DESC
LIMIT 20;
```

## Summary

ClickHouse enables cloud utilization analysis at scale by storing high-frequency metric samples from cloud providers and returning aggregations in milliseconds. Identify idle and over-provisioned resources, compute right-sizing candidates, and visualize utilization heatmaps by time of day. These insights can reduce cloud spend by 20-40% through targeted right-sizing initiatives.
