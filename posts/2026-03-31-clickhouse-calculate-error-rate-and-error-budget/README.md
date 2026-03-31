# How to Calculate Error Rate and Error Budget in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Error Rate, Error Budget, SLO, Observability, Analytics

Description: Learn how to calculate error rate and error budget in ClickHouse using SQL aggregations to power SLO dashboards and reliability tracking.

---

Error rate and error budget are two of the most critical reliability metrics for any production system. ClickHouse is well-suited for computing these at scale because of its columnar storage and fast aggregation capabilities.

## What Is Error Rate?

Error rate is the percentage of requests that result in an error within a time window. It is typically computed as:

```text
error_rate = error_count / total_count * 100
```

## What Is Error Budget?

An error budget is the allowed amount of unreliability under an SLO. If your SLO is 99.9% availability, your error budget is 0.1% of requests or time.

## Schema Setup

Assume you have a table tracking HTTP requests:

```sql
CREATE TABLE http_requests (
    timestamp DateTime,
    service String,
    status_code UInt16,
    duration_ms UInt32
) ENGINE = MergeTree()
ORDER BY (service, timestamp);
```

## Calculating Error Rate

```sql
SELECT
    service,
    toStartOfHour(timestamp) AS hour,
    countIf(status_code >= 500) AS error_count,
    count() AS total_count,
    round(countIf(status_code >= 500) / count() * 100, 4) AS error_rate_pct
FROM http_requests
WHERE timestamp >= now() - INTERVAL 24 HOUR
GROUP BY service, hour
ORDER BY service, hour;
```

## Calculating Error Budget Remaining

Assuming a 99.9% SLO (0.1% budget):

```sql
WITH
    0.001 AS allowed_error_rate,
    countIf(status_code >= 500) AS errors,
    count() AS total
SELECT
    service,
    total,
    errors,
    round((1 - errors / total) * 100, 4) AS success_rate_pct,
    round((allowed_error_rate - (errors / total)) / allowed_error_rate * 100, 2) AS budget_remaining_pct
FROM http_requests
WHERE timestamp >= now() - INTERVAL 30 DAY
GROUP BY service;
```

## Using a Materialized View for Continuous Tracking

For production use, pre-aggregate error stats into a summary table:

```sql
CREATE MATERIALIZED VIEW error_rate_mv
ENGINE = SummingMergeTree()
ORDER BY (service, hour)
AS
SELECT
    service,
    toStartOfHour(timestamp) AS hour,
    countIf(status_code >= 500) AS error_count,
    count() AS total_count
FROM http_requests
GROUP BY service, hour;
```

Then query the view for fast SLO dashboards:

```sql
SELECT
    service,
    hour,
    sum(error_count) AS errors,
    sum(total_count) AS total,
    round(sum(error_count) / sum(total_count) * 100, 4) AS error_rate_pct
FROM error_rate_mv
WHERE hour >= now() - INTERVAL 7 DAY
GROUP BY service, hour
ORDER BY service, hour;
```

## Burn Rate Alerting

Burn rate measures how quickly you are consuming your error budget. A burn rate of 1x means you are consuming it at the expected rate; 2x means you will exhaust it in half the time.

```sql
SELECT
    service,
    round(sum(error_count) / sum(total_count) / 0.001, 2) AS burn_rate
FROM error_rate_mv
WHERE hour >= now() - INTERVAL 1 HOUR
GROUP BY service
HAVING burn_rate > 2;
```

## Summary

ClickHouse makes it straightforward to calculate error rate and error budget using its native SQL aggregation functions. By combining raw event tables with materialized views, you can power real-time SLO dashboards, burn rate alerts, and budget tracking with minimal latency and infrastructure overhead.
