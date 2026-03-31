# How to Use ClickHouse with Census for Reverse ETL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Census, Reverse ETL, Data Activation, CRM

Description: Use Census with ClickHouse as the source to sync computed user segments and metrics from your warehouse into CRMs, ad platforms, and other tools.

---

## What Is Census

Census is a reverse ETL platform that reads data from your data warehouse and syncs it to operational tools like Salesforce, HubSpot, Marketo, Facebook Ads, and hundreds of other destinations. It handles incremental syncs, schema mapping, and error handling so you do not need to build custom integrations.

## Connecting ClickHouse to Census

In the Census dashboard, add ClickHouse as a source:

1. Go to Sources and click Add Source
2. Select ClickHouse
3. Configure the connection:

```text
Host: ch.internal
Port: 9440 (native TLS)
Database: analytics
Username: census_reader
Password: <secret>
```

Grant the census user read access to relevant tables:

```sql
CREATE USER IF NOT EXISTS census_reader
    IDENTIFIED WITH sha256_password BY 'secret'
    HOST IP '10.0.0.0/8';

GRANT SELECT ON analytics.* TO census_reader;
```

## Building a Source SQL Model

Census uses SQL models to define what data to sync. Create a model for high-value customer segments:

```sql
SELECT
    u.user_id,
    u.email,
    u.company_name,
    sum(o.total_cents) / 100.0 AS lifetime_value_usd,
    count(DISTINCT o.order_id) AS order_count,
    max(o.created_at) AS last_order_date,
    CASE
        WHEN sum(o.total_cents) > 100000 THEN 'enterprise'
        WHEN sum(o.total_cents) > 10000 THEN 'mid_market'
        ELSE 'smb'
    END AS segment
FROM analytics.users u
JOIN analytics.orders o ON o.user_id = u.user_id
WHERE o.status = 'completed'
GROUP BY u.user_id, u.email, u.company_name
```

## Syncing to Salesforce

Configure a sync from the model to Salesforce Contacts:

```text
Source: ClickHouse model (above)
Destination: Salesforce Contact
Match on: Email -> Email
Fields to sync:
  lifetime_value_usd -> LTV__c
  order_count -> Total_Orders__c
  segment -> Customer_Segment__c
  last_order_date -> Last_Purchase_Date__c
Schedule: Every 4 hours
```

Census handles upserts automatically - records are created if they do not exist and updated if they do.

## Syncing Audiences to Facebook Ads

Push user segments to Facebook Custom Audiences for targeted ad campaigns:

```sql
-- Census model for retargeting
SELECT email, user_id
FROM analytics.users
WHERE last_seen_at < now() - INTERVAL 30 DAY
  AND last_seen_at > now() - INTERVAL 90 DAY
  AND total_orders > 0
```

## Monitoring Sync Status

Census provides a sync history dashboard. For additional monitoring, log sync results to ClickHouse itself:

```sql
CREATE TABLE analytics.census_sync_log (
    sync_time    DateTime DEFAULT now(),
    model_name   String,
    destination  LowCardinality(String),
    records_synced UInt32,
    records_failed UInt32,
    status       LowCardinality(String)
) ENGINE = MergeTree()
ORDER BY sync_time;
```

Alert via OneUptime when `records_failed > 0`.

## Summary

Census with ClickHouse as the source warehouse enables automated reverse ETL that pushes computed insights, segments, and metrics from ClickHouse into CRMs, ad platforms, and operational tools - closing the loop between analytics and action.
