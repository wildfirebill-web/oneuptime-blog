# How to Handle Data Sovereignty with ClickHouse Multi-Region

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Data Sovereignty, Compliance, Multi-Region, GDPR

Description: Configure ClickHouse to keep user data within specific geographic boundaries using partitioning, storage policies, and query routing for regulatory compliance.

---

Data sovereignty laws (GDPR, data localization) require that certain user data never leaves a specific country or region. ClickHouse supports this through partition-based data placement and storage policies tied to region-specific object storage buckets.

## Schema Design for Data Sovereignty

Add a `data_region` column to every sensitive table and use it as a partition key:

```sql
CREATE TABLE user_events
(
    user_id     UInt64,
    data_region LowCardinality(String),  -- 'EU', 'US', 'APAC'
    payload     String,
    ts          DateTime
)
ENGINE = ReplicatedMergeTree(...)
PARTITION BY (data_region, toYYYYMM(ts))
ORDER BY (user_id, ts);
```

## Region-Specific Storage Policies

Define separate S3 buckets per sovereignty zone:

```xml
<disks>
    <s3_eu>
        <type>s3</type>
        <endpoint>https://s3.eu-central-1.amazonaws.com/ch-eu-data/</endpoint>
    </s3_eu>
    <s3_us>
        <type>s3</type>
        <endpoint>https://s3.us-east-1.amazonaws.com/ch-us-data/</endpoint>
    </s3_us>
</disks>
<policies>
    <eu_only><volumes><v><disk>s3_eu</disk></v></volumes></eu_only>
    <us_only><volumes><v><disk>s3_us</disk></v></volumes></us_only>
</policies>
```

Create separate tables per region with matching storage policies:

```sql
CREATE TABLE user_events_eu ... SETTINGS storage_policy = 'eu_only';
CREATE TABLE user_events_us ... SETTINGS storage_policy = 'us_only';
```

## Restricting Cross-Region Queries

Use row-level policies to prevent EU data from being returned to non-EU query sources:

```sql
CREATE ROW POLICY eu_only ON user_events
    USING data_region = 'EU'
    TO eu_analysts;
```

## Auditing Data Access

```sql
SELECT
    user,
    query_start_time,
    query,
    read_rows
FROM system.query_log
WHERE type = 'QueryFinish'
  AND has(tables, 'user_events')
ORDER BY query_start_time DESC;
```

Export this log to your compliance SIEM system daily.

## Monitoring and Compliance Alerting

Set up [OneUptime](https://oneuptime.com) scheduled checks that verify no EU-tagged rows appear in the US storage bucket. Alert the compliance team immediately if a cross-boundary data leak is detected.

## Summary

ClickHouse handles data sovereignty through partition-based placement, region-scoped storage policies, and row-level access controls. Pair these with query audit logs to demonstrate compliance to regulators.
