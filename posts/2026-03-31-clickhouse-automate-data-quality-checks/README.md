# How to Automate ClickHouse Data Quality Checks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Data Quality, Automation, Validation, Monitoring

Description: Automate ClickHouse data quality checks with SQL assertions and scheduled scripts to catch bad data before it reaches dashboards or ML models.

---

## Why Data Quality Automation Matters

Silent data quality issues corrupt analytics and erode trust. Automated checks run continuously and alert your team the moment data drifts outside expected bounds, rather than waiting for a user to notice a wrong number.

## Categories of Data Quality Checks

There are four main categories to cover:

1. Completeness - are expected rows arriving?
2. Validity - are values within allowed ranges?
3. Uniqueness - are primary keys distinct?
4. Timeliness - is recent data actually recent?

## Completeness Check

Verify that the events table receives a minimum number of rows each hour:

```sql
SELECT
    toStartOfHour(event_time) AS hour,
    count() AS row_count
FROM events
WHERE event_time >= now() - INTERVAL 25 HOUR
  AND event_time < now() - INTERVAL 1 HOUR
GROUP BY hour
HAVING row_count < 10000
ORDER BY hour;
```

If this query returns rows, those hours have suspiciously low data volume.

## Validity Check

Check for out-of-range values in numeric columns:

```sql
SELECT count() AS invalid_rows
FROM orders
WHERE total_amount < 0
   OR total_amount > 1000000
   OR user_id = 0;
```

## Uniqueness Check

Detect duplicate primary keys in a daily partition:

```sql
SELECT order_id, count() AS cnt
FROM orders
WHERE toDate(created_at) = today()
GROUP BY order_id
HAVING cnt > 1
LIMIT 10;
```

## Timeliness Check

Ensure the most recent record is not older than 10 minutes:

```sql
SELECT
    max(event_time) AS latest,
    dateDiff('minute', max(event_time), now()) AS lag_minutes
FROM events;
```

## Automating Checks with a Script

Wrap assertions in a shell script that returns a non-zero exit code on failure:

```bash
#!/bin/bash
set -e

INVALID=$(clickhouse-client --query \
  "SELECT count() FROM orders WHERE total_amount < 0" --format TSVRaw)

if [ "${INVALID}" -gt 0 ]; then
  echo "ERROR: Found ${INVALID} rows with negative total_amount"
  exit 1
fi

LAG=$(clickhouse-client --query \
  "SELECT dateDiff('minute', max(event_time), now()) FROM events" --format TSVRaw)

if [ "${LAG}" -gt 10 ]; then
  echo "ERROR: Event data is ${LAG} minutes stale"
  exit 1
fi

echo "All data quality checks passed"
```

## Integrating with Great Expectations

For a framework-based approach, use the ClickHouse SQLAlchemy connector with Great Expectations:

```text
datasource:
  class_name: SqlAlchemyDatasource
  credentials:
    url: clickhouse+native://user:pass@host:9000/analytics
```

Then define expectations as YAML and run them in CI or on a schedule.

## Alerting on Failures

Route failures to your on-call channel via OneUptime or PagerDuty so the data engineering team can investigate before dashboards show bad numbers.

## Summary

Automating ClickHouse data quality checks with SQL assertions, shell scripts, and alerting ensures bad data is caught early, protecting downstream analytics and model training pipelines from silent corruption.
