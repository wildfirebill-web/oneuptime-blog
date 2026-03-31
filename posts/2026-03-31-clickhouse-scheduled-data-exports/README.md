# How to Set Up Scheduled Data Exports from ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Scheduled Export, Automation, Cron, Data Pipeline

Description: Learn how to set up scheduled data exports from ClickHouse using cron jobs, ClickHouse Scheduled Jobs, and pipeline tools for automated reporting.

---

Scheduled exports automate the delivery of ClickHouse data to downstream systems - dashboards, data warehouses, or object storage - on a regular cadence.

## Option 1 - Cron Jobs

The simplest approach uses a shell script triggered by cron:

```bash
#!/bin/bash
# /opt/scripts/export_events.sh
set -e

DATE=$(date -d yesterday +%Y-%m-%d)
EXPORT_PATH="/exports/events_${DATE}.csv.gz"

clickhouse-client \
  --host "${CLICKHOUSE_HOST}" \
  --user default \
  --password "${CLICKHOUSE_PASSWORD}" \
  --query "SELECT * FROM events WHERE toDate(ts) = '${DATE}'" \
  --format CSVWithNames \
  | gzip > "${EXPORT_PATH}"

# Upload to S3
aws s3 cp "${EXPORT_PATH}" "s3://my-bucket/events/date=${DATE}/data.csv.gz"
rm "${EXPORT_PATH}"

echo "Export complete for ${DATE}"
```

Register with cron:

```bash
# Run daily at 2am
0 2 * * * /opt/scripts/export_events.sh >> /var/log/clickhouse-exports.log 2>&1
```

## Option 2 - ClickHouse Scheduled Jobs

ClickHouse 24.5+ supports `CREATE SCHEDULE`:

```sql
CREATE SCHEDULE daily_events_export
    CRON '0 2 * * *'
AS
    INSERT INTO FUNCTION s3(
        'https://s3.amazonaws.com/my-bucket/events/{toDate(now()-1)}/data.parquet',
        'Parquet'
    )
    SELECT * FROM events
    WHERE toDate(ts) = today() - 1;
```

## Option 3 - Materialized View with Background Refresh

For continuously updated export targets:

```sql
CREATE TABLE events_daily_summary (
    date Date,
    event_type LowCardinality(String),
    event_count UInt64,
    unique_users UInt32
) ENGINE = SummingMergeTree()
ORDER BY (date, event_type);

CREATE MATERIALIZED VIEW events_daily_mv TO events_daily_summary
AS SELECT
    toDate(ts) AS date,
    event_type,
    count() AS event_count,
    uniq(user_id) AS unique_users
FROM events
GROUP BY date, event_type;
```

## Option 4 - Airflow DAG

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime, timedelta

with DAG('clickhouse_export', schedule_interval='@daily', start_date=datetime(2026,1,1)) as dag:
    export = BashOperator(
        task_id='export_to_s3',
        bash_command="""
        clickhouse-client --query "
          INSERT INTO FUNCTION s3('s3://my-bucket/events/{{ ds }}/data.parquet', 'Parquet')
          SELECT * FROM events WHERE toDate(ts) = '{{ ds }}'
        "
        """
    )
```

## Summary

Schedule ClickHouse exports via cron for simplicity, ClickHouse Scheduled Jobs for self-contained automation, or Airflow for orchestrated pipelines. Use materialized views with SummingMergeTree for continuously refreshed aggregated exports.
