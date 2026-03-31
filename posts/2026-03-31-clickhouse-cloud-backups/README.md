# How to Configure ClickHouse Cloud Backups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ClickHouse Cloud, Backup, Restore, Disaster Recovery, Data Protection

Description: Learn how ClickHouse Cloud handles automated backups, how to restore from a backup, and how to configure backup retention for your services.

---

ClickHouse Cloud automatically backs up your data with no additional configuration required. Understanding how backups work, what the retention policy is, and how to trigger restores helps you plan your disaster recovery and compliance strategy.

## Automatic Backup Schedule

ClickHouse Cloud performs automated backups:
- **Full backups**: Daily
- **Incremental backups**: Every few hours (exact schedule depends on service tier)
- **Retention**: 1 day for Development tier, 7 days for Production tier by default

These are managed entirely by ClickHouse Cloud - you do not need to configure backup jobs.

## Viewing Available Backups

In the ClickHouse Cloud console:
1. Open your service
2. Click "Backups"
3. View the list of available restore points with timestamps and sizes

Via API:

```bash
curl https://api.clickhouse.cloud/v1/organizations/${ORG_ID}/services/${SERVICE_ID}/backups \
  -H "Authorization: Bearer ${CLICKHOUSE_API_KEY}" \
  | jq '.result[] | {id: .id, status: .status, createdAt: .createdAt, size: .sizeInBytes}'
```

## Restoring from a Backup

Restores create a **new service** - they do not overwrite your existing service:

```bash
curl -X POST \
  https://api.clickhouse.cloud/v1/organizations/${ORG_ID}/services/${SERVICE_ID}/backups/${BACKUP_ID}/restore \
  -H "Authorization: Bearer ${CLICKHOUSE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "serviceName": "analytics-restored",
    "region": "us-east-1",
    "provider": "aws"
  }'
```

## Exporting Data as a Self-Managed Backup

For additional protection or to maintain your own backups:

```sql
-- Export to S3
INSERT INTO FUNCTION s3(
  's3://my-backups/clickhouse/events/2024-03-31.parquet',
  'KEY', 'SECRET',
  'Parquet'
)
SELECT * FROM analytics.events
WHERE event_date = today();
```

## Checking Backup Size

```sql
SELECT
    database,
    formatReadableSize(sum(bytes_on_disk)) AS total_size
FROM system.parts
WHERE active = 1
GROUP BY database;
```

## Point-in-Time Recovery

ClickHouse Cloud does not currently support arbitrary point-in-time recovery. Restores are to specific backup snapshots. For fine-grained PITR, maintain your own write-ahead log via Kafka or use the S3 export pattern above.

## Summary

ClickHouse Cloud handles automated daily and incremental backups with configurable retention. Restore to a new service via the console or API. Supplement managed backups with S3 exports for cross-account redundancy or extended retention beyond the 7-day default.
