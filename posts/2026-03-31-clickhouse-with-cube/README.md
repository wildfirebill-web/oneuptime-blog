# How to Use WITH CUBE in ClickHouse for Cross-Tabulation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SELECT, GROUP BY, WITH CUBE, Cross-Tabulation

Description: Generate all grouping combinations at once in ClickHouse using GROUP BY WITH CUBE, producing cross-tabulated subtotals across every dimension simultaneously.

---

`WITH CUBE` extends `GROUP BY` to produce every possible combination of the listed columns, including the grand total. For `GROUP BY a, b, c WITH CUBE`, ClickHouse computes all 2^3 = 8 grouping combinations: `(a,b,c)`, `(a,b)`, `(a,c)`, `(b,c)`, `(a)`, `(b)`, `(c)`, and `()`. This is the multidimensional equivalent of `WITH ROLLUP` - instead of a single hierarchy it delivers a full cross-tabulation cube, which is the foundation of OLAP pivot tables.

## Basic Syntax

```sql
SELECT
    col1,
    col2,
    aggregate_function(col3) AS metric
FROM table_name
GROUP BY col1, col2 WITH CUBE;
```

## Two-Dimensional Cube Example

```sql
CREATE TABLE ad_spend
(
    channel  String,
    region   String,
    spend    Float64
)
ENGINE = MergeTree()
ORDER BY (channel, region);

INSERT INTO ad_spend VALUES
    ('search', 'US', 500),
    ('search', 'EU', 300),
    ('social', 'US', 400),
    ('social', 'EU', 200);

SELECT
    channel,
    region,
    sum(spend) AS spend
FROM ad_spend
GROUP BY channel, region WITH CUBE
ORDER BY channel NULLS LAST, region NULLS LAST;
```

```text
channel | region | spend
--------|--------|------
search  | EU     | 300    <- (channel, region)
search  | US     | 500    <- (channel, region)
search  | NULL   | 800    <- (channel) total
social  | EU     | 200    <- (channel, region)
social  | US     | 400    <- (channel, region)
social  | NULL   | 600    <- (channel) total
NULL    | EU     | 500    <- (region) total
NULL    | US     | 900    <- (region) total
NULL    | NULL   | 1400   <- grand total
```

With 2 dimensions the cube has 4 distinct grouping sets (2^2). Three dimensions would produce 8 sets, four dimensions 16, and so on.

## Three-Dimensional Cube

```sql
SELECT
    year,
    channel,
    region,
    sum(spend)  AS spend,
    count()     AS campaigns
FROM marketing_data
GROUP BY year, channel, region WITH CUBE
ORDER BY
    year    NULLS LAST,
    channel NULLS LAST,
    region  NULLS LAST;
```

This returns 2^3 = 8 grouping combinations - every subset of `{year, channel, region}`.

## Using GROUPING() to Label Cube Rows

`NULL` appears in any column that was collapsed for a given grouping set. Use `GROUPING()` to distinguish rollup-generated `NULL`s from real `NULL` data:

```sql
SELECT
    if(GROUPING(channel) = 1, 'ALL CHANNELS', channel) AS channel_label,
    if(GROUPING(region)  = 1, 'ALL REGIONS',  region)  AS region_label,
    sum(spend)                                           AS spend,
    GROUPING(channel)                                    AS g_channel,
    GROUPING(region)                                     AS g_region
FROM ad_spend
GROUP BY channel, region WITH CUBE
ORDER BY channel NULLS LAST, region NULLS LAST;
```

## Filtering for Specific Grouping Sets

You can use `HAVING` with `GROUPING()` to retrieve only the grouping sets you care about:

```sql
-- Only cross-dimension totals (one dimension collapsed, one not)
SELECT
    channel,
    region,
    sum(spend) AS spend
FROM ad_spend
GROUP BY channel, region WITH CUBE
HAVING (GROUPING(channel) + GROUPING(region)) = 1
ORDER BY channel NULLS LAST, region NULLS LAST;
```

```text
channel | region | spend
--------|--------|------
search  | NULL   | 800
social  | NULL   | 600
NULL    | EU     | 500
NULL    | US     | 900
```

## WITH CUBE vs. WITH ROLLUP

| Feature | WITH ROLLUP | WITH CUBE |
|---|---|---|
| Grouping sets produced | N+1 (hierarchical) | 2^N (all combinations) |
| Row count growth | Linear | Exponential |
| Use case | Hierarchy subtotals (year > quarter > month) | Cross-tabulation across independent dimensions |

```sql
-- ROLLUP: 3 grouping levels for (channel, region)
SELECT channel, region, sum(spend)
FROM ad_spend
GROUP BY channel, region WITH ROLLUP;
-- Groupings: (channel,region), (channel), ()

-- CUBE: 4 grouping levels for (channel, region)
SELECT channel, region, sum(spend)
FROM ad_spend
GROUP BY channel, region WITH CUBE;
-- Groupings: (channel,region), (channel), (region), ()
```

## Practical Report - Marketing Spend Pivot

```sql
SELECT
    if(GROUPING(year)    = 1, 'Total', toString(year))    AS year_label,
    if(GROUPING(channel) = 1, 'Total', channel)           AS channel_label,
    sum(spend)                                             AS spend
FROM marketing_data
GROUP BY year, channel WITH CUBE
ORDER BY year NULLS LAST, channel NULLS LAST;
```

## Summary

`WITH CUBE` generates all 2^N grouping combinations from the listed `GROUP BY` columns in a single pass, enabling full cross-tabulation without multiple queries or application-side pivoting. Use `GROUPING()` to tell apart synthetic `NULL` subtotal markers from actual `NULL` values in your data. Prefer `WITH ROLLUP` when your dimensions are naturally hierarchical; reach for `WITH CUBE` when every combination of dimensions is independently meaningful.
