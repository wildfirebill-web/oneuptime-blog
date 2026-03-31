# ClickHouse for DynamoDB Developers - Key Differences

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, DynamoDB, AWS, Analytics, Migration

Description: A guide for DynamoDB developers learning ClickHouse, covering key differences in data model, querying, cost model, and analytics use cases.

---

DynamoDB is AWS's managed NoSQL database optimized for key-value and document access with single-digit millisecond latency at any scale. ClickHouse is a columnar analytical database optimized for aggregate queries over billions of rows. Teams often use both: DynamoDB for their application's primary data store and ClickHouse for analytics.

## Access Pattern Philosophy

DynamoDB enforces query-driven data modeling. You must define a partition key and sort key upfront and cannot run arbitrary queries efficiently. ClickHouse supports arbitrary SQL queries with the sorting key as a performance hint, not a hard constraint.

```sql
-- DynamoDB: you must know the partition key
# Only efficient if you know user_id
response = table.query(
    KeyConditionExpression=Key('user_id').eq('u123')
)

-- ClickHouse: scan the entire table efficiently
SELECT
    country,
    count() AS users,
    avg(lifetime_value) AS avg_ltv
FROM users
WHERE created_at >= '2024-01-01'
GROUP BY country
ORDER BY avg_ltv DESC;
```

## Schema Flexibility

DynamoDB is schemaless - any item can have any attributes. ClickHouse requires a defined schema but supports semi-structured data via JSON or Map types:

```sql
CREATE TABLE user_events (
    event_time DateTime,
    user_id String,
    event_type LowCardinality(String),
    properties Map(String, String)
) ENGINE = MergeTree()
ORDER BY (user_id, event_time);

-- Query map keys
SELECT
    properties['page'] AS page,
    count() AS views
FROM user_events
WHERE event_type = 'page_view'
GROUP BY page
ORDER BY views DESC;
```

## Cost Model

DynamoDB charges per read/write capacity unit or on-demand per request. Analytical queries that scan large tables are extremely expensive on DynamoDB. ClickHouse charges based on compute and storage, making full-table analytical queries economical.

A query that scans 10 billion DynamoDB rows would cost thousands of dollars per run. The same query on ClickHouse costs cents in compute time.

## DynamoDB Streams to ClickHouse

A common pattern is to stream DynamoDB changes to ClickHouse using DynamoDB Streams and Lambda:

```python
import json
import clickhouse_connect

ch = clickhouse_connect.get_client(
    host='your-clickhouse-host',
    port=8443,
    secure=True
)

def lambda_handler(event, context):
    rows = []
    for record in event['Records']:
        if record['eventName'] in ('INSERT', 'MODIFY'):
            new_image = record['dynamodb']['NewImage']
            rows.append([
                new_image['user_id']['S'],
                new_image['event_type']['S'],
                new_image['timestamp']['N'],
            ])
    if rows:
        ch.insert('user_events', rows,
                  column_names=['user_id', 'event_type', 'event_time'])
```

## Global Tables vs. Replication

DynamoDB Global Tables provide multi-region active-active replication automatically. ClickHouse replication uses `ReplicatedMergeTree` within a region; cross-region replication requires additional setup.

## Summary

DynamoDB developers moving to ClickHouse gain the ability to run analytical SQL queries across all data without worrying about access patterns or cost per scan. The combination of DynamoDB for operational workloads and ClickHouse for analytics is a common and effective architecture for AWS-based applications.
