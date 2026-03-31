# How to Track Material Consumption Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Material Consumption, Manufacturing, Bill of Materials, Cost Analytics

Description: Track material consumption against bill of materials targets in ClickHouse to identify waste, variance, and yield loss across production runs.

---

Material consumption tracking compares actual material usage against standard bill of materials (BOM) quantities to identify waste, over-consumption, and yield losses. ClickHouse aggregates high-volume production consumption records efficiently.

## Material Consumption Table

```sql
CREATE TABLE material_consumption (
    consumption_id UUID,
    plant_id       LowCardinality(String),
    work_order_id  String,
    material_code  LowCardinality(String),
    material_name  String,
    material_type  LowCardinality(String),  -- raw_material, component, packaging
    quantity_used  Float64,
    quantity_std   Float64,  -- standard BOM quantity
    unit_cost      Decimal64(4),
    unit           LowCardinality(String),
    consumed_at    DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(consumed_at)
ORDER BY (plant_id, material_code, consumed_at);
```

## Material Variance by Work Order

Compare actual vs. standard consumption per work order:

```sql
SELECT
    work_order_id,
    plant_id,
    material_code,
    material_name,
    sum(quantity_std) AS std_quantity,
    sum(quantity_used) AS actual_quantity,
    sum(quantity_used) - sum(quantity_std) AS variance_qty,
    round((sum(quantity_used) - sum(quantity_std)) / nullIf(sum(quantity_std), 0) * 100, 2) AS variance_pct,
    sum((quantity_used - quantity_std) * unit_cost) AS variance_cost
FROM material_consumption
WHERE consumed_at >= today() - 30
GROUP BY work_order_id, plant_id, material_code, material_name
HAVING abs(variance_pct) > 5
ORDER BY variance_cost DESC;
```

## High-Waste Materials

Find materials with consistently high over-consumption:

```sql
SELECT
    material_code,
    material_name,
    material_type,
    count(DISTINCT work_order_id) AS work_orders,
    avg((quantity_used - quantity_std) / nullIf(quantity_std, 0) * 100) AS avg_variance_pct,
    sum((quantity_used - quantity_std) * unit_cost) AS total_waste_cost
FROM material_consumption
WHERE consumed_at >= today() - 90
GROUP BY material_code, material_name, material_type
HAVING avg_variance_pct > 3
ORDER BY total_waste_cost DESC;
```

## Daily Material Cost vs. Budget

Track actual material spend against standard cost:

```sql
SELECT
    plant_id,
    toDate(consumed_at) AS day,
    material_type,
    sum(quantity_used * unit_cost) AS actual_cost,
    sum(quantity_std * unit_cost) AS standard_cost,
    sum(quantity_used * unit_cost) - sum(quantity_std * unit_cost) AS cost_variance
FROM material_consumption
WHERE consumed_at >= today() - 30
GROUP BY plant_id, day, material_type
ORDER BY plant_id, day;
```

## Scrap and Rework Material Cost

Estimate cost of materials consumed in defective products:

```sql
SELECT
    material_code,
    material_name,
    toStartOfMonth(consumed_at) AS month,
    sumIf(quantity_used * unit_cost, quantity_used > quantity_std * 1.1) AS estimated_scrap_cost
FROM material_consumption
WHERE consumed_at >= today() - 90
GROUP BY material_code, material_name, month
ORDER BY estimated_scrap_cost DESC;
```

## Material Turnover Rate

Measure how quickly materials are consumed:

```sql
SELECT
    material_code,
    material_name,
    sum(quantity_used) AS total_consumed_90d,
    avg(quantity_used) AS avg_daily_consumption,
    -- assumes inventory table available
    count(DISTINCT work_order_id) AS work_orders
FROM material_consumption
WHERE consumed_at >= today() - 90
GROUP BY material_code, material_name
ORDER BY total_consumed_90d DESC
LIMIT 50;
```

## Summary

ClickHouse enables detailed material consumption analytics - tracking BOM variance, identifying high-waste materials, measuring daily cost vs. standard, and estimating scrap costs. These insights drive material efficiency programs and reduce cost of goods sold.
