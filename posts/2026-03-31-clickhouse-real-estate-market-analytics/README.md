# How to Use ClickHouse for Real Estate Market Analytics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Real Estate, Market Analytics, Property Data, Geospatial

Description: Build real estate market analytics with ClickHouse to track price trends, neighborhood comparisons, days on market, and investment return metrics.

---

## Real Estate Analytics at Scale

Real estate platforms accumulate hundreds of millions of property listing records, transaction histories, and market data points. ClickHouse enables fast analytics across this data for market trend reports, investment analysis, and automated valuation models.

## Property Transactions Schema

```sql
CREATE TABLE property_transactions
(
    transaction_id UInt64,
    close_date Date,
    mls_number String,
    property_type LowCardinality(String),
    city LowCardinality(String),
    state LowCardinality(String),
    zip_code LowCardinality(String),
    neighborhood LowCardinality(String),
    sale_price Decimal(12, 2),
    list_price Decimal(12, 2),
    square_footage UInt32,
    bedrooms UInt8,
    bathrooms Float32,
    year_built UInt16,
    days_on_market UInt16,
    latitude Float64,
    longitude Float64
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(close_date)
ORDER BY (state, city, zip_code, close_date);
```

## Price Trend Analysis

```sql
-- Median price per square foot by neighborhood over time
SELECT
    neighborhood,
    toStartOfMonth(close_date) AS month,
    median(sale_price / square_footage) AS median_price_per_sqft,
    count() AS sales_count,
    median(days_on_market) AS median_dom
FROM property_transactions
WHERE state = 'CA'
  AND property_type = 'Single Family'
  AND close_date >= today() - 365
GROUP BY neighborhood, month
ORDER BY neighborhood, month;
```

## Market Heat Index

```sql
-- Identify hot markets (sale price > list price, low DOM)
SELECT
    zip_code,
    city,
    count() AS sales_volume,
    avg(sale_price / list_price - 1) * 100 AS avg_overbid_pct,
    median(days_on_market) AS median_dom,
    median(sale_price) AS median_sale_price
FROM property_transactions
WHERE close_date >= today() - 90
  AND property_type IN ('Single Family', 'Condo')
GROUP BY zip_code, city
HAVING sales_volume >= 10
ORDER BY avg_overbid_pct DESC, median_dom ASC
LIMIT 20;
```

## Year-Over-Year Appreciation

```sql
WITH current_year AS (
    SELECT zip_code, median(sale_price) AS median_price
    FROM property_transactions
    WHERE toYear(close_date) = toYear(today())
    GROUP BY zip_code
),
prior_year AS (
    SELECT zip_code, median(sale_price) AS median_price
    FROM property_transactions
    WHERE toYear(close_date) = toYear(today()) - 1
    GROUP BY zip_code
)
SELECT
    c.zip_code,
    c.median_price AS current_median,
    p.median_price AS prior_median,
    round((c.median_price - p.median_price) / p.median_price * 100, 1) AS yoy_appreciation_pct
FROM current_year AS c
JOIN prior_year AS p ON c.zip_code = p.zip_code
ORDER BY yoy_appreciation_pct DESC
LIMIT 30;
```

## Investment Return Analysis

```sql
-- Price-to-rent ratio by neighborhood
CREATE TABLE rental_listings
(
    listing_date Date,
    neighborhood LowCardinality(String),
    property_type LowCardinality(String),
    bedrooms UInt8,
    monthly_rent Decimal(8, 2)
)
ENGINE = MergeTree
ORDER BY (neighborhood, property_type, listing_date);
```

```sql
SELECT
    t.neighborhood,
    median(t.sale_price) AS median_sale_price,
    median(r.monthly_rent) * 12 AS median_annual_rent,
    round(median(t.sale_price) / (median(r.monthly_rent) * 12), 1) AS price_to_rent_ratio
FROM property_transactions AS t
JOIN rental_listings AS r
    ON t.neighborhood = r.neighborhood
    AND t.bedrooms = r.bedrooms
WHERE t.close_date >= today() - 180
  AND r.listing_date >= today() - 90
GROUP BY t.neighborhood
ORDER BY price_to_rent_ratio ASC
LIMIT 20;
```

## Summary

ClickHouse powers real estate market analytics with fast aggregations over millions of property records. Use the MergeTree engine with partition by month and sort by location hierarchy, then build queries for price trends, market heat indexes, year-over-year appreciation, and investment metrics. The columnar engine handles all typical real estate analytical workloads in milliseconds.
