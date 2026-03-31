# How to Build Data Reconciliation Queries in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Data Reconciliation, Data Quality, Aggregation, Audit, Comparison

Description: Build data reconciliation queries in ClickHouse to compare row counts, sums, and checksums between source systems and ClickHouse to verify data completeness.

---

## What Is Data Reconciliation?

Data reconciliation verifies that data loaded into ClickHouse matches the source system. Common checks:
- Row counts match between source and ClickHouse
- Aggregate sums (revenue, event counts) agree
- No duplicate rows were inserted
- No rows were dropped during ETL

## Basic Count Reconciliation

Compare ClickHouse row counts against expected counts from a control table:

```sql
SELECT
    expected.date,
    expected.count AS expected_count,
    actual.count   AS actual_count,
    expected.count - actual.count AS discrepancy
FROM (
    SELECT toDate(ts) AS date, row_count AS count
    FROM etl_control_table
    WHERE date >= today() - 7
) expected
LEFT JOIN (
    SELECT toDate(ts) AS date, count() AS count
    FROM events
    WHERE ts >= today() - 7
    GROUP BY date
) actual USING (date)
ORDER BY date DESC;
```

Flag any non-zero discrepancies for investigation.

## Sum Reconciliation

For financial or business-critical totals:

```sql
SELECT
    toDate(order_date) AS day,
    sum(amount)        AS ch_total
FROM orders
WHERE order_date >= today() - 7
GROUP BY day
ORDER BY day DESC;
```

Compare this output against totals from the source OLTP database. Any difference indicates insert failures or double-counting.

## Deduplication Check

Detect duplicate rows (same business key inserted multiple times):

```sql
SELECT
    order_id,
    count() AS count
FROM orders
WHERE order_date >= today()
GROUP BY order_id
HAVING count > 1
ORDER BY count DESC
LIMIT 50;
```

If `ReplacingMergeTree` is used, query with `FINAL`:

```sql
SELECT count() FROM orders FINAL WHERE order_date >= today();
```

## Checksum-Based Reconciliation

Generate a checksum over a dataset for comparison:

```sql
SELECT
    toDate(ts)        AS day,
    count()           AS row_count,
    sum(cityHash64(order_id, amount, user_id)) AS checksum
FROM orders
WHERE ts >= today() - 1
GROUP BY day;
```

If the checksum matches between source and ClickHouse, the data is identical.

## Partition-Level Reconciliation

Run reconciliation per partition to isolate issues:

```sql
SELECT
    partition,
    sum(rows)                     AS total_rows,
    formatReadableSize(sum(bytes_on_disk)) AS disk_size,
    count()                       AS part_count
FROM system.parts
WHERE database = 'default' AND table = 'orders' AND active
GROUP BY partition
ORDER BY partition DESC
LIMIT 10;
```

Compare with source system partition totals.

## Automating Reconciliation

Schedule reconciliation queries and store results:

```sql
INSERT INTO reconciliation_results
SELECT
    now()          AS check_time,
    toDate(ts)     AS data_date,
    count()        AS row_count,
    sum(amount)    AS total_amount
FROM orders
WHERE ts >= today() - 1
GROUP BY data_date;
```

Alert when the stored totals deviate from expected values.

## Summary

Build ClickHouse data reconciliation queries by comparing row counts, aggregate sums, and checksums against source system data, detecting duplicates with GROUP BY HAVING, and scheduling automated checks that store results for trending. Partition-level reconciliation isolates issues to specific time windows for faster root cause analysis.
