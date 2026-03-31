# How to Analyze Supplier Performance Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Supplier, Performance, Procurement, Analytics, Quality

Description: Analyze supplier performance metrics in ClickHouse - on-time delivery, defect rates, lead time variability, and price compliance across your supplier base.

---

Supplier performance analytics helps procurement teams identify reliable partners, manage risk, and negotiate better contracts. ClickHouse aggregates purchase order and receipt data to surface actionable metrics.

## Purchase Order and Receipt Tables

```sql
CREATE TABLE purchase_orders (
    po_id        String,
    supplier_id  LowCardinality(String),
    supplier_name String,
    sku          String,
    qty_ordered  UInt32,
    unit_price   Decimal64(2),
    promised_date Date,
    category     LowCardinality(String),
    created_at   DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (supplier_id, created_at);

CREATE TABLE po_receipts (
    receipt_id   String,
    po_id        String,
    supplier_id  LowCardinality(String),
    qty_received UInt32,
    qty_rejected UInt32,
    actual_date  Date,
    received_at  DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(received_at)
ORDER BY (supplier_id, received_at);
```

## On-Time In-Full (OTIF) Rate

```sql
SELECT
    po.supplier_id,
    po.supplier_name,
    count() AS total_pos,
    countIf(r.actual_date <= po.promised_date
        AND r.qty_received >= po.qty_ordered) AS otif_count,
    round(countIf(r.actual_date <= po.promised_date
        AND r.qty_received >= po.qty_ordered) / count() * 100, 2) AS otif_pct
FROM purchase_orders po
JOIN po_receipts r ON po.po_id = r.po_id
WHERE po.created_at >= today() - 90
GROUP BY po.supplier_id, po.supplier_name
ORDER BY otif_pct;
```

## Defect and Rejection Rate

```sql
SELECT
    supplier_id,
    sum(qty_received) AS total_received,
    sum(qty_rejected) AS total_rejected,
    round(sum(qty_rejected) / nullIf(sum(qty_received), 0) * 100, 3) AS rejection_rate_pct
FROM po_receipts
WHERE received_at >= today() - 90
GROUP BY supplier_id
HAVING rejection_rate_pct > 2
ORDER BY rejection_rate_pct DESC;
```

## Lead Time Analysis

Measure actual vs. promised lead times:

```sql
SELECT
    po.supplier_id,
    po.supplier_name,
    avg(dateDiff('day', toDate(po.created_at), r.actual_date)) AS avg_lead_days,
    avg(dateDiff('day', toDate(po.created_at), po.promised_date)) AS avg_promised_days,
    quantile(0.9)(dateDiff('day', toDate(po.created_at), r.actual_date)) AS p90_lead_days,
    stddevPop(dateDiff('day', toDate(po.created_at), r.actual_date)) AS lead_time_std
FROM purchase_orders po
JOIN po_receipts r ON po.po_id = r.po_id
WHERE po.created_at >= today() - 180
GROUP BY po.supplier_id, po.supplier_name
ORDER BY avg_lead_days DESC;
```

## Price Compliance

Compare invoice prices against contracted rates:

```sql
SELECT
    supplier_id,
    sku,
    avg(unit_price) AS avg_invoiced_price,
    -- assumes contract_price stored separately
    count() AS po_count,
    sum(qty_ordered * unit_price) AS total_spend
FROM purchase_orders
WHERE created_at >= today() - 90
GROUP BY supplier_id, sku
ORDER BY total_spend DESC
LIMIT 100;
```

## Supplier Scorecard

Composite score combining OTIF, quality, and lead time:

```sql
SELECT
    supplier_id,
    otif_pct,
    100 - rejection_rate_pct AS quality_score,
    100 - least(avg_lead_deviation * 5, 100) AS lead_time_score,
    round((otif_pct + (100 - rejection_rate_pct) + (100 - least(avg_lead_deviation * 5, 100))) / 3, 2) AS composite_score
FROM (
    SELECT
        po.supplier_id,
        round(countIf(r.actual_date <= po.promised_date AND r.qty_received >= po.qty_ordered) / count() * 100, 2) AS otif_pct,
        round(sum(r.qty_rejected) / nullIf(sum(r.qty_received), 0) * 100, 3) AS rejection_rate_pct,
        avg(abs(dateDiff('day', po.promised_date, r.actual_date))) AS avg_lead_deviation
    FROM purchase_orders po
    JOIN po_receipts r ON po.po_id = r.po_id
    WHERE po.created_at >= today() - 90
    GROUP BY po.supplier_id
)
ORDER BY composite_score DESC;
```

## Summary

ClickHouse enables procurement teams to compute comprehensive supplier scorecards from purchase order and receipt data. OTIF rates, defect percentages, lead time variability, and price compliance can all be aggregated quickly across thousands of suppliers and SKUs.
