# How to Build Pricing Tier Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Pricing Analytics, SaaS, Plan Usage, Upgrade Signal

Description: Learn how to analyze usage by pricing tier in ClickHouse to identify upgrade candidates, over-quota accounts, and plan fit signals.

---

Pricing tier analytics answers questions that drive SaaS revenue: "which free-tier users are near their limits?", "which pro accounts are using less than 20% of their quota?", "what plan do most churned users come from?". ClickHouse makes it easy to combine usage events with subscription metadata for these analyses.

## Schema

```sql
CREATE TABLE plan_usage_events
(
    ts           DateTime,
    account_id   UInt64,
    plan         LowCardinality(String), -- 'free','starter','pro','enterprise'
    resource     LowCardinality(String), -- 'api_calls','storage_gb','seats','events'
    quantity     Float64
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (account_id, ts);

CREATE TABLE plan_limits
(
    plan          LowCardinality(String),
    resource      LowCardinality(String),
    monthly_limit Float64
)
ENGINE = MergeTree()
ORDER BY (plan, resource);
```

## Usage vs Quota per Account

```sql
SELECT
    u.account_id,
    u.plan,
    u.resource,
    sum(u.quantity)        AS used,
    any(l.monthly_limit)   AS limit,
    sum(u.quantity) * 100.0 / any(l.monthly_limit) AS pct_used
FROM plan_usage_events u
JOIN plan_limits l ON u.plan = l.plan AND u.resource = l.resource
WHERE toStartOfMonth(u.ts) = toStartOfMonth(now())
GROUP BY u.account_id, u.plan, u.resource
ORDER BY pct_used DESC
LIMIT 50;
```

## Accounts Near Their Limit (Upgrade Candidates)

```sql
-- accounts above 80% of their monthly quota
SELECT account_id, plan, resource, pct_used
FROM (
    SELECT
        u.account_id,
        u.plan,
        u.resource,
        sum(u.quantity) * 100.0 / any(l.monthly_limit) AS pct_used
    FROM plan_usage_events u
    JOIN plan_limits l ON u.plan = l.plan AND u.resource = l.resource
    WHERE toStartOfMonth(u.ts) = toStartOfMonth(now())
    GROUP BY u.account_id, u.plan, u.resource
)
WHERE pct_used >= 80
  AND plan != 'enterprise'
ORDER BY pct_used DESC;
```

## Usage Distribution by Plan

```sql
SELECT
    plan,
    resource,
    quantile(0.5)(quantity_30d) AS median_usage,
    quantile(0.9)(quantity_30d) AS p90_usage
FROM (
    SELECT account_id, plan, resource, sum(quantity) AS quantity_30d
    FROM plan_usage_events
    WHERE ts >= now() - INTERVAL 30 DAY
    GROUP BY account_id, plan, resource
)
GROUP BY plan, resource
ORDER BY plan, resource;
```

## Under-Utilization - Downgrade Risk

```sql
SELECT account_id, plan, resource, quantity_30d
FROM (
    SELECT account_id, plan, resource, sum(quantity) AS quantity_30d
    FROM plan_usage_events
    WHERE ts >= now() - INTERVAL 30 DAY
    GROUP BY account_id, plan, resource
)
JOIN plan_limits l ON plan = l.plan AND resource = l.resource
WHERE quantity_30d < l.monthly_limit * 0.20
  AND plan IN ('pro', 'enterprise')
ORDER BY quantity_30d ASC;
```

## Revenue by Plan - Monthly Trend

```sql
SELECT
    toStartOfMonth(ts)  AS month,
    plan,
    uniqExact(account_id) AS active_accounts
FROM plan_usage_events
GROUP BY month, plan
ORDER BY month, plan;
```

## Summary

ClickHouse simplifies pricing tier analytics by joining usage event streams with plan limit metadata. Identify accounts approaching their quota as upgrade candidates, flag under-utilizing enterprise accounts as downgrade risks, and track plan distribution trends month-over-month. These insights help align pricing strategy with actual customer usage patterns.
