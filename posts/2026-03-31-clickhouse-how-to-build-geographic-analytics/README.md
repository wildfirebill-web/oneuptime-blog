# How to Build Geographic Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Geographic Analytics, Geospatial, Analytics, SQL

Description: Learn how to build geographic analytics in ClickHouse using geospatial functions and efficient data models for location-based insights.

---

Geographic analytics helps you understand where your users are, how regional trends differ, and which markets drive the most value. ClickHouse provides solid geospatial support that makes it practical to run location queries at scale.

## Setting Up a Geographic Events Table

Start with a table that stores user events alongside location data.

```sql
CREATE TABLE geo_events
(
    event_id UUID DEFAULT generateUUIDv4(),
    user_id UInt64,
    event_type LowCardinality(String),
    country_code FixedString(2),
    region String,
    city String,
    latitude Float64,
    longitude Float64,
    event_time DateTime,
    revenue Decimal(10, 2)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (country_code, event_time, user_id);
```

## Country-Level Aggregations

Aggregate revenue and sessions per country for dashboard widgets.

```sql
SELECT
    country_code,
    count() AS sessions,
    countDistinct(user_id) AS unique_users,
    sum(revenue) AS total_revenue,
    avg(revenue) AS avg_revenue_per_session
FROM geo_events
WHERE event_time >= now() - INTERVAL 30 DAY
GROUP BY country_code
ORDER BY total_revenue DESC
LIMIT 20;
```

## City-Level Funnel Analysis

Identify which cities have the highest conversion rates.

```sql
SELECT
    city,
    country_code,
    countIf(event_type = 'page_view') AS page_views,
    countIf(event_type = 'purchase') AS purchases,
    round(
        countIf(event_type = 'purchase') * 100.0 /
        nullIf(countIf(event_type = 'page_view'), 0),
        2
    ) AS conversion_rate_pct
FROM geo_events
WHERE event_time >= now() - INTERVAL 7 DAY
GROUP BY city, country_code
HAVING page_views > 100
ORDER BY conversion_rate_pct DESC
LIMIT 30;
```

## Proximity Queries Using Haversine Distance

Find all events within 50 km of a reference point (e.g., a store location).

```sql
SELECT
    user_id,
    city,
    latitude,
    longitude,
    greatCircleDistance(latitude, longitude, 40.7128, -74.0060) / 1000 AS dist_km
FROM geo_events
WHERE event_time >= today()
HAVING dist_km < 50
ORDER BY dist_km ASC;
```

## Time-Zone Aware Reporting

Adjust event times to local time zones for each region.

```sql
SELECT
    country_code,
    toStartOfHour(toTimeZone(event_time, 'America/New_York')) AS local_hour,
    count() AS events
FROM geo_events
WHERE country_code = 'US'
  AND event_time >= now() - INTERVAL 24 HOUR
GROUP BY country_code, local_hour
ORDER BY local_hour;
```

## Materialized View for Region Summaries

Pre-aggregate region-level stats to speed up dashboards.

```sql
CREATE MATERIALIZED VIEW region_daily_summary
ENGINE = SummingMergeTree()
ORDER BY (country_code, region, day)
AS
SELECT
    country_code,
    region,
    toDate(event_time) AS day,
    count() AS events,
    sum(revenue) AS revenue
FROM geo_events
GROUP BY country_code, region, day;
```

Query the summary view for fast regional breakdowns:

```sql
SELECT region, sum(events) AS total_events, sum(revenue) AS total_revenue
FROM region_daily_summary
WHERE day >= today() - 30
GROUP BY region
ORDER BY total_revenue DESC;
```

## Summary

ClickHouse handles geographic analytics well through its geospatial functions, `LowCardinality` types for country and region fields, and materialized views for pre-aggregated summaries. Partitioning by month and ordering by country plus time keeps queries fast even across billions of rows. Combining proximity queries with `greatCircleDistance` and time-zone conversions gives you a complete toolkit for building location-aware dashboards.
