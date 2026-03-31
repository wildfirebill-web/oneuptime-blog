# How to Build Invoice Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Invoice, Analytics, Finance, Accounts Receivable, Aging

Description: Build invoice analytics in ClickHouse to track accounts receivable aging, payment delays, DSO, and overdue invoice trends by customer and region.

---

Invoice analytics powers accounts receivable management - identifying overdue invoices, calculating Days Sales Outstanding (DSO), and flagging customers with chronic late payments. ClickHouse handles large invoice datasets efficiently.

## Invoice Table

```sql
CREATE TABLE invoices (
    invoice_id    UUID,
    customer_id   UInt64,
    customer_name String,
    region        LowCardinality(String),
    currency      LowCardinality(String),
    amount        Decimal64(2),
    tax_amount    Decimal64(2),
    total_amount  Decimal64(2),
    issue_date    Date,
    due_date      Date,
    paid_date     Nullable(Date),
    status        LowCardinality(String)  -- draft, issued, paid, overdue, disputed
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(issue_date)
ORDER BY (customer_id, issue_date);
```

## Accounts Receivable Aging Buckets

Classify outstanding invoices into aging buckets:

```sql
SELECT
    customer_id,
    customer_name,
    countIf(today() - due_date BETWEEN 0 AND 30) AS current_count,
    sumIf(total_amount, today() - due_date BETWEEN 0 AND 30) AS current_amount,
    countIf(today() - due_date BETWEEN 31 AND 60) AS days_31_60_count,
    sumIf(total_amount, today() - due_date BETWEEN 31 AND 60) AS days_31_60_amount,
    countIf(today() - due_date BETWEEN 61 AND 90) AS days_61_90_count,
    sumIf(total_amount, today() - due_date BETWEEN 61 AND 90) AS days_61_90_amount,
    countIf(today() - due_date > 90) AS over_90_count,
    sumIf(total_amount, today() - due_date > 90) AS over_90_amount
FROM invoices
WHERE status IN ('issued', 'overdue')
GROUP BY customer_id, customer_name
ORDER BY over_90_amount DESC;
```

## Days Sales Outstanding (DSO)

Calculate DSO for each month - a key receivables KPI:

```sql
SELECT
    toStartOfMonth(issue_date) AS month,
    sum(total_amount) AS total_invoiced,
    avg(dateDiff('day', issue_date, paid_date)) AS avg_days_to_pay,
    quantile(0.9)(dateDiff('day', issue_date, paid_date)) AS p90_days_to_pay
FROM invoices
WHERE status = 'paid'
  AND paid_date IS NOT NULL
  AND issue_date >= today() - 365
GROUP BY month
ORDER BY month;
```

## Late Payment Rate by Customer

Identify chronic late payers:

```sql
SELECT
    customer_id,
    customer_name,
    count() AS total_invoices,
    countIf(paid_date > due_date) AS late_payments,
    round(countIf(paid_date > due_date) / count() * 100, 2) AS late_rate_pct,
    avg(dateDiff('day', due_date, paid_date)) AS avg_days_late
FROM invoices
WHERE status = 'paid'
  AND issue_date >= today() - 365
GROUP BY customer_id, customer_name
HAVING late_rate_pct > 30
ORDER BY avg_days_late DESC;
```

## Overdue Invoice Trend

Track the value of overdue invoices over time:

```sql
SELECT
    toStartOfWeek(issue_date) AS week,
    countIf(status = 'overdue') AS overdue_count,
    sumIf(total_amount, status = 'overdue') AS overdue_value,
    sumIf(total_amount, status = 'paid') AS collected_value
FROM invoices
WHERE issue_date >= today() - 90
GROUP BY week
ORDER BY week;
```

## Collection Rate by Region

Measure how effectively each region collects invoices:

```sql
SELECT
    region,
    sum(total_amount) AS total_invoiced,
    sumIf(total_amount, status = 'paid') AS total_collected,
    round(sumIf(total_amount, status = 'paid') / sum(total_amount) * 100, 2) AS collection_rate_pct
FROM invoices
WHERE issue_date >= today() - 180
GROUP BY region
ORDER BY collection_rate_pct;
```

## Summary

ClickHouse simplifies invoice analytics with fast aggregations across large receivables datasets. Aging bucket analysis, DSO calculation, late payment identification, and collection rate tracking are all expressible in clean SQL, making ClickHouse an effective backend for financial operations dashboards.
