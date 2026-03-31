# How to Use ClickHouse with Metabase

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Metabase, Business Intelligence, Analytics, Visualization

Description: Learn how to connect Metabase to ClickHouse using the official ClickHouse driver, build questions, create dashboards, and use Metabase's SQL editor for advanced analytics.

---

> Metabase's official ClickHouse driver provides a native connection that supports both the GUI question builder and raw SQL for self-service analytics.

Metabase is one of the most popular open-source BI tools for non-technical users. The official ClickHouse Metabase driver enables full integration: browsing tables, building no-code questions, creating dashboards, and writing raw SQL queries. This guide covers installation, connection setup, and building effective analytics.

---

## Installing the ClickHouse Driver

Download and configure the ClickHouse Metabase plugin.

```bash
# Create the plugins directory
mkdir -p /opt/metabase/plugins

# Download the ClickHouse Metabase driver
curl -L \
  https://github.com/ClickHouse/metabase-clickhouse-driver/releases/download/1.3.3/clickhouse.metabase-driver.jar \
  -o /opt/metabase/plugins/clickhouse.metabase-driver.jar

# Verify the file
ls -lh /opt/metabase/plugins/
```

## Running Metabase with Docker

Launch Metabase with the ClickHouse driver mounted.

```bash
docker run -d \
  --name metabase \
  -p 3000:3000 \
  -e MB_DB_TYPE=h2 \
  -e JAVA_OPTS="-Xmx2g" \
  -v /opt/metabase/plugins:/plugins \
  -v /opt/metabase/data:/metabase-data \
  metabase/metabase:latest
```

```yaml
# docker-compose.yml
version: "3.8"

services:
  metabase:
    image: metabase/metabase:latest
    ports:
      - "3000:3000"
    environment:
      MB_DB_FILE: /metabase-data/metabase.db
      JAVA_OPTS: "-Xmx2g"
    volumes:
      - ./plugins:/plugins
      - metabase_data:/metabase-data
    depends_on:
      - clickhouse

  clickhouse:
    image: clickhouse/clickhouse-server:latest
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - ch_data:/var/lib/clickhouse

volumes:
  metabase_data:
  ch_data:
```

## Connecting Metabase to ClickHouse

Configure the database connection through the Metabase UI.

```text
Metabase -> Admin -> Databases -> Add Database

Database Type:  ClickHouse
Name:           ClickHouse Analytics
Host:           localhost  (or your ClickHouse host)
Port:           8123
Database:       default
Username:       metabase_user
Password:       metabase_password

Advanced Options:
  Use a secure connection (SSL): Yes (for production)
  Additional JDBC arguments: (leave blank to start)
```

## Creating a Read-Only Metabase User in ClickHouse

Restrict Metabase's access to SELECT-only.

```sql
-- Create a dedicated Metabase user
CREATE USER metabase_user
IDENTIFIED WITH plaintext_password BY 'metabase_password'
HOST ANY;

-- Grant SELECT on analytics tables
GRANT SELECT ON analytics.* TO metabase_user;

-- Allow reading system columns for schema introspection
GRANT SELECT ON system.columns   TO metabase_user;
GRANT SELECT ON system.tables    TO metabase_user;
GRANT SELECT ON system.databases TO metabase_user;
```

## Creating ClickHouse Tables for Metabase Dashboards

Build well-structured tables that work well with Metabase's question builder.

```sql
CREATE DATABASE IF NOT EXISTS analytics;

CREATE TABLE analytics.website_traffic
(
    date        Date,
    page        String,
    country     LowCardinality(String),
    device      LowCardinality(String),
    source      LowCardinality(String),
    sessions    UInt32,
    pageviews   UInt32,
    bounces     UInt32,
    duration_s  Float32
)
ENGINE = SummingMergeTree((sessions, pageviews, bounces, duration_s))
PARTITION BY toYYYYMM(date)
ORDER BY (date, country, device, source, page);

-- Populate with sample data
INSERT INTO analytics.website_traffic
SELECT
    today() - (number % 90)                                  AS date,
    concat('/page-', toString(number % 30))                  AS page,
    ['US','GB','DE','CA','AU'][1 + number % 5]               AS country,
    ['desktop','mobile','tablet'][1 + number % 3]            AS device,
    ['organic','paid','direct','social'][1 + number % 4]     AS source,
    rand() % 200 + 10                                        AS sessions,
    rand() % 600 + 30                                        AS pageviews,
    rand() % 100                                             AS bounces,
    round(randCanonical() * 300, 1)                          AS duration_s
FROM numbers(50000);
```

## Building Questions with the GUI

Metabase's question builder maps naturally to ClickHouse queries.

```text
Metabase -> New Question -> Simple Question

Table: analytics.website_traffic

Filters:
  date -> Current Quarter
  device -> is not -> tablet

Summarize:
  Sum of sessions grouped by: date (by Week), country

Sort: date ascending

Visualization: Line chart
  X-axis: date
  Y-axis: Sum of sessions
  Series: country
```

## Writing SQL Questions for ClickHouse

Use ClickHouse-specific functions in Metabase SQL queries.

```sql
-- Top pages by traffic trend (last 30 days vs previous 30 days)
WITH
    current_period AS (
        SELECT
            page,
            sum(sessions) AS current_sessions
        FROM analytics.website_traffic
        WHERE date >= today() - 30
        GROUP BY page
    ),
    previous_period AS (
        SELECT
            page,
            sum(sessions) AS previous_sessions
        FROM analytics.website_traffic
        WHERE date >= today() - 60
          AND date <  today() - 30
        GROUP BY page
    )
SELECT
    c.page,
    c.current_sessions,
    p.previous_sessions,
    round(
        (c.current_sessions - p.previous_sessions)
        / p.previous_sessions * 100,
        1
    ) AS growth_pct
FROM current_period c
LEFT JOIN previous_period p USING (page)
WHERE p.previous_sessions > 0
ORDER BY current_sessions DESC
LIMIT 20;
```

## Creating a Dashboard

Assemble multiple questions into a ClickHouse analytics dashboard.

```text
Metabase -> New Dashboard -> ClickHouse Analytics

Add cards:
  1. "Sessions Over Time" - Line chart (daily sessions, last 90 days)
  2. "Traffic by Country" - Map visualization
  3. "Device Split" - Pie chart (desktop vs mobile vs tablet)
  4. "Top Pages" - Table with sparklines
  5. "Bounce Rate Trend" - Line chart

Add dashboard filters:
  - Date Range filter -> connected to all cards (date column)
  - Country filter    -> connected to country-aware cards

Auto-refresh: Every 60 minutes
```

## Configuring ClickHouse for Metabase Performance

Add optimizations for Metabase query patterns.

```sql
-- Create a projection for common Metabase filter patterns
ALTER TABLE analytics.website_traffic
    ADD PROJECTION p_by_country
    (
        SELECT date, country, device, source,
               sum(sessions), sum(pageviews), sum(bounces)
        GROUP BY date, country, device, source
        ORDER BY country, date
    );

-- Materialize the projection
ALTER TABLE analytics.website_traffic
    MATERIALIZE PROJECTION p_by_country;
```

```xml
<!-- /etc/clickhouse-server/users.d/metabase_profile.xml -->
<clickhouse>
    <profiles>
        <metabase>
            <max_execution_time>30</max_execution_time>
            <max_rows_to_read>100000000</max_rows_to_read>
            <use_query_cache>1</use_query_cache>
            <query_cache_ttl>300</query_cache_ttl>
        </metabase>
    </profiles>
    <users>
        <metabase_user>
            <profile>metabase</profile>
        </metabase_user>
    </users>
</clickhouse>
```

## Summary

Metabase connects to ClickHouse through the official ClickHouse driver JAR. Install the driver in the plugins directory, configure the HTTP connection, and restrict the Metabase user to SELECT-only permissions. Use `SummingMergeTree` for pre-aggregated tables that power dashboard cards. Metabase's GUI question builder works well for standard aggregations, while the SQL editor gives access to ClickHouse-specific functions like `WITH` CTEs, window functions, and array operations. Add projections to speed up common dashboard filter patterns.
