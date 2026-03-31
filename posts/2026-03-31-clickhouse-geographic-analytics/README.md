# How to Build Geographic Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Geographic Analytics, GeoIP, Geospatial, Country Analytics, Web Analytics

Description: Learn how to build geographic analytics in ClickHouse using GeoIP enrichment, country and city analysis, and geospatial query patterns.

---

Geographic analytics lets you understand where your users come from, which regions drive the most traffic, and how behavior varies by geography. ClickHouse supports both IP-based geo-enrichment and native geospatial operations.

## Storing Geographic Data

Enrich events with geo data at ingestion time using a GeoIP service or ClickHouse's built-in dictionaries:

```sql
CREATE TABLE web_events (
    ts         DateTime64(3),
    visitor_id String,
    ip_address IPv4,
    country    LowCardinality(String),  -- ISO 3166-1 alpha-2
    region     LowCardinality(String),
    city       String,
    latitude   Float32,
    longitude  Float32,
    timezone   LowCardinality(String),
    url        String
) ENGINE = MergeTree
PARTITION BY toYYYYMMDD(ts)
ORDER BY (country, ts);
```

## IP-to-Country Lookup with Dictionaries

Use a ClickHouse dictionary backed by a GeoIP database for fast IP resolution:

```sql
CREATE DICTIONARY geoip_country (
    prefix      String,
    country_iso String,
    country     String
)
PRIMARY KEY prefix
SOURCE(HTTP(URL 'https://cdn.db-ip.com/db/csv/latest/db-ip-country-lite-ipv4.csv.gz' FORMAT CSV))
LAYOUT(IP_TRIE)
LIFETIME(86400);

-- Use in queries
SELECT
    dictGetString('geoip_country', 'country', tuple(ip_address)) AS country,
    count() AS visits
FROM web_events
GROUP BY country
ORDER BY visits DESC;
```

## Country Traffic Analysis

```sql
SELECT
    country,
    uniq(visitor_id)     AS unique_visitors,
    count()              AS pageviews,
    avg(session_pages)   AS avg_pages_per_session
FROM (
    SELECT
        visitor_id,
        session_id,
        any(country)     AS country,
        count()          AS session_pages
    FROM web_events
    WHERE ts >= today() - 30
    GROUP BY visitor_id, session_id
)
GROUP BY country
ORDER BY unique_visitors DESC
LIMIT 20;
```

## City-Level Analysis

```sql
SELECT
    country,
    city,
    count()          AS pageviews,
    uniq(visitor_id) AS visitors
FROM web_events
WHERE ts >= today() - 7
  AND country = 'US'
GROUP BY country, city
ORDER BY visitors DESC
LIMIT 20;
```

## Regional Heatmap Data

Generate data for a world heatmap visualization:

```sql
SELECT
    country,
    latitude,
    longitude,
    count()          AS events
FROM web_events
WHERE ts >= today() - 7
  AND latitude != 0
GROUP BY country, latitude, longitude;
```

## Timezone-Aware Traffic Analysis

```sql
SELECT
    timezone,
    toHour(toTimeZone(ts, timezone))   AS local_hour,
    count()                            AS pageviews
FROM web_events
WHERE ts >= today() - 7
  AND timezone != ''
GROUP BY timezone, local_hour
ORDER BY timezone, local_hour;
```

## Geographic Conversion Rates

```sql
SELECT
    country,
    uniq(visitor_id)                             AS visitors,
    uniqIf(visitor_id, event_type = 'purchase')  AS buyers,
    round(100.0 *
        uniqIf(visitor_id, event_type = 'purchase') /
        uniq(visitor_id), 2)                     AS conversion_rate_pct
FROM web_events
WHERE ts >= today() - 30
GROUP BY country
HAVING visitors > 100
ORDER BY conversion_rate_pct DESC
LIMIT 20;
```

## Summary

ClickHouse's LowCardinality columns for country and region data, combined with IP_TRIE dictionary layout for fast GeoIP lookups, make geographic analytics both fast and storage-efficient. The patterns here support country/city dashboards, regional conversion analysis, and timezone-aware traffic studies all from a single events table.
