# How to Use S3 Tiering to Reduce ClickHouse Infrastructure Costs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, S3, Storage Tiering, Cost Optimization, Cold Storage

Description: Use ClickHouse storage policies with S3 tiering to automatically move old data to cheap object storage while keeping hot data on fast local disks.

---

## The Cost Problem with Single-Tier Storage

Keeping all data on NVMe SSDs is expensive. In many analytics workloads, 80% of queries touch only the most recent 30 days of data. Older data sits on costly SSDs rarely accessed. S3 tiering solves this by automatically moving cold data to object storage at a fraction of the cost.

## Configure S3 Storage in ClickHouse

Add S3 disk and storage policy to `config.xml` or a file under `config.d/`:

```xml
<storage_configuration>
  <disks>
    <local_fast>
      <type>local</type>
      <path>/var/lib/clickhouse/</path>
    </local_fast>
    <s3_cold>
      <type>s3</type>
      <endpoint>https://s3.amazonaws.com/my-clickhouse-bucket/cold/</endpoint>
      <access_key_id>ACCESS_KEY</access_key_id>
      <secret_access_key>SECRET_KEY</secret_access_key>
      <region>us-east-1</region>
    </s3_cold>
  </disks>

  <policies>
    <hot_to_cold>
      <volumes>
        <hot>
          <disk>local_fast</disk>
          <max_data_part_size_bytes>10737418240</max_data_part_size_bytes>
        </hot>
        <cold>
          <disk>s3_cold</disk>
        </cold>
      </volumes>
      <move_factor>0.2</move_factor>
    </hot_to_cold>
  </policies>
</storage_configuration>
```

## Create a Table with the Tiering Policy

```sql
CREATE TABLE events
(
    event_time DateTime,
    user_id UInt64,
    event_type LowCardinality(String),
    properties String
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_type, event_time)
TTL event_time + INTERVAL 30 DAY TO VOLUME 'cold'
SETTINGS storage_policy = 'hot_to_cold';
```

The TTL rule automatically moves parts older than 30 days to S3.

## Manually Trigger Part Movement

If you want to move parts immediately rather than waiting for the TTL background process:

```sql
ALTER TABLE events MOVE PARTITION '202401' TO VOLUME 'cold';
```

## Check Where Data Lives

```sql
SELECT
    partition,
    disk_name,
    formatReadableSize(sum(bytes_on_disk)) AS size_on_disk
FROM system.parts
WHERE table = 'events' AND active
GROUP BY partition, disk_name
ORDER BY partition;
```

## Cost Estimate

For 10TB of data with 2TB accessed in the last 30 days:

```text
Without tiering: 10TB * $0.10/GB/month SSD = $1,024/month
With tiering:    2TB  * $0.10/GB/month SSD  = $204/month
                 8TB  * $0.023/GB/month S3   = $188/month
Total:           $392/month  (62% savings)
```

## Performance Considerations

Queries hitting S3 are slower due to network latency. To minimize impact:
- Always filter on partition keys to avoid scanning cold data unnecessarily
- Use the `prefer_fetch_column_from_remote` setting carefully
- Consider keeping a summary materialized view on local disk for fast aggregations over historical data

## Summary

S3 tiering in ClickHouse lets you define storage policies that automatically migrate old partitions from fast local SSDs to inexpensive S3 buckets using TTL rules. The result is typically 50-70% storage cost reduction with minimal impact on query performance for recent data.
