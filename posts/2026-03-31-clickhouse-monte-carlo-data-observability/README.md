# How to Use ClickHouse with Monte Carlo for Data Observability

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Monte Carlo, Data Observability, Data Quality, Monitoring

Description: Connect Monte Carlo to ClickHouse to automatically detect data anomalies, track freshness and volume, and get alerted when your data breaks.

---

## What Is Data Observability

Data observability means monitoring the health of your data in the same way you monitor the health of your services. Monte Carlo is a leading data observability platform that connects to your data warehouse and automatically detects anomalies in table freshness, volume, schema, and distributions.

## Connecting Monte Carlo to ClickHouse

Monte Carlo connects to ClickHouse via its metadata and query APIs. In the Monte Carlo dashboard:

1. Go to Settings > Data Stores > Add Data Store
2. Select ClickHouse
3. Enter connection details:

```text
Host: ch.internal
Port: 8123
Database: analytics
Username: montecarlo_reader
Password: <secret>
```

Create a read-only service account with access to system tables:

```sql
CREATE USER montecarlo_reader
    IDENTIFIED WITH sha256_password BY 'mc_secret'
    HOST IP '34.123.0.0/16';  -- Monte Carlo IP range

GRANT SELECT ON analytics.* TO montecarlo_reader;
GRANT SELECT ON system.tables TO montecarlo_reader;
GRANT SELECT ON system.columns TO montecarlo_reader;
GRANT SELECT ON system.parts TO montecarlo_reader;
GRANT SELECT ON system.query_log TO montecarlo_reader;
```

## What Monte Carlo Monitors Automatically

Once connected, Monte Carlo automatically monitors:

- **Freshness** - when each table was last updated
- **Volume** - row count changes over time
- **Schema** - column additions, deletions, and type changes
- **Distribution** - changes in null rates, unique values, and value distributions

## Custom Monitors for ClickHouse

Define custom SQL monitors for business-logic checks:

```sql
-- Monitor: orders should arrive every hour
SELECT
    toStartOfHour(created_at) AS hour,
    count() AS order_count
FROM analytics.orders
WHERE created_at >= now() - INTERVAL 48 HOUR
GROUP BY hour
ORDER BY hour;
```

Monte Carlo detects when `order_count` drops below its historical baseline.

## Lineage Tracking

Monte Carlo parses ClickHouse query logs to build a lineage graph automatically. Enable query log access:

```sql
GRANT SELECT ON system.query_log TO montecarlo_reader;
```

This allows Monte Carlo to trace data flow from source tables through materialized views to reporting tables.

## Incident Response with Circuit Breakers

Configure Monte Carlo circuit breakers to pause downstream pipelines when data quality issues are detected:

```text
If: orders freshness > 2 hours
Then: pause dbt production run
      send alert to #data-alerts Slack channel
      create incident in PagerDuty
```

## Comparing Monte Carlo with Manual Checks

| Capability | Manual SQL checks | Monte Carlo |
|-----------|------------------|-------------|
| Freshness monitoring | Manual cron | Automatic |
| Anomaly detection | Fixed thresholds | ML-based baselines |
| Schema change detection | Manual review | Automatic |
| Lineage | Manual mapping | Automatic from query log |

## Summary

Monte Carlo with ClickHouse provides automatic data observability without requiring you to write and maintain custom monitoring SQL - it learns normal patterns from your data and alerts you when anomalies occur, reducing the time to detect data incidents.
