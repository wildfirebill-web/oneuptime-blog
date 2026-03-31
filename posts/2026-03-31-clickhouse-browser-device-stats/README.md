# How to Track Browser and Device Statistics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Web Analytics, Browser, Device, User Agent, Analytics

Description: Track browser, device, and operating system statistics in ClickHouse by storing parsed user agent data and querying it for audience segmentation and compatibility analysis.

---

Understanding which browsers and devices your users use drives compatibility testing decisions and performance optimization priorities. ClickHouse stores and queries this data efficiently at any scale.

## Schema Design

Parse user agents at ingestion time and store structured fields rather than raw strings:

```sql
CREATE TABLE page_events (
    session_id   String,
    user_id      UInt64,
    path         LowCardinality(String),
    browser      LowCardinality(String),
    browser_ver  LowCardinality(String),
    os           LowCardinality(String),
    os_ver       LowCardinality(String),
    device_type  LowCardinality(String),
    is_mobile    UInt8,
    is_bot       UInt8,
    created_at   DateTime DEFAULT now()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, session_id);
```

## Parsing User Agents Before Insert

Use a server-side library to parse the user agent before inserting into ClickHouse. Example in Python with `ua-parser`:

```python
from ua_parser import user_agent_parser
import clickhouse_connect

client = clickhouse_connect.get_client(host='localhost')

def parse_and_insert(session_id, user_id, path, raw_ua):
    parsed = user_agent_parser.Parse(raw_ua)
    ua = parsed['user_agent']
    os = parsed['os']
    device = parsed['device']

    client.insert('page_events', [[
        session_id, user_id, path,
        ua['family'],
        ua['major'] or '',
        os['family'],
        os['major'] or '',
        device['family'],
        1 if device['family'] in ('iPhone','Android','iPad') else 0,
        0
    ]])
```

## Browser Market Share

```sql
SELECT
    browser,
    uniq(session_id) AS sessions,
    round(100 * sessions / sum(sessions) OVER (), 2) AS share_pct
FROM page_events
WHERE created_at >= today() - 30
  AND is_bot = 0
GROUP BY browser
ORDER BY sessions DESC
LIMIT 10;
```

## Browser Version Distribution

```sql
SELECT
    browser,
    browser_ver,
    uniq(session_id) AS sessions
FROM page_events
WHERE browser = 'Chrome'
  AND created_at >= today() - 30
  AND is_bot = 0
GROUP BY browser, browser_ver
ORDER BY sessions DESC
LIMIT 10;
```

## Mobile vs Desktop

```sql
SELECT
    multiIf(is_mobile = 1, 'Mobile', 'Desktop') AS device_category,
    uniq(session_id) AS sessions,
    round(100 * sessions / sum(sessions) OVER (), 2) AS share_pct
FROM page_events
WHERE created_at >= today() - 30
  AND is_bot = 0
GROUP BY device_category;
```

## Operating System Distribution

```sql
SELECT
    os,
    uniq(session_id) AS sessions,
    round(100 * sessions / sum(sessions) OVER (), 2) AS share_pct
FROM page_events
WHERE created_at >= today() - 30
  AND is_bot = 0
GROUP BY os
ORDER BY sessions DESC
LIMIT 10;
```

## Device Type Breakdown

```sql
SELECT
    device_type,
    count() AS events,
    uniq(session_id) AS sessions,
    uniq(user_id) AS users
FROM page_events
WHERE created_at >= today() - 30
  AND is_bot = 0
GROUP BY device_type
ORDER BY sessions DESC;
```

## Materialized View for Daily Browser Stats

```sql
CREATE TABLE browser_daily (
    date     Date,
    browser  LowCardinality(String),
    os       LowCardinality(String),
    is_mobile UInt8,
    sessions UInt64
) ENGINE = SummingMergeTree(sessions)
ORDER BY (date, browser, os, is_mobile);

CREATE MATERIALIZED VIEW browser_daily_mv
TO browser_daily AS
SELECT
    toDate(created_at) AS date,
    browser,
    os,
    is_mobile,
    uniq(session_id) AS sessions
FROM page_events
WHERE is_bot = 0
GROUP BY date, browser, os, is_mobile;
```

## Trend: Mobile Share Over Time

```sql
SELECT
    date,
    sum(sessions) AS total_sessions,
    sumIf(sessions, is_mobile = 1) AS mobile_sessions,
    round(100 * mobile_sessions / total_sessions, 2) AS mobile_pct
FROM browser_daily
WHERE date >= today() - 90
GROUP BY date
ORDER BY date;
```

## Summary

Storing pre-parsed browser and device attributes in ClickHouse rather than raw user agent strings enables fast segmentation queries. The SummingMergeTree materialized view pattern gives you a daily rollup table for trend dashboards without full table scans at query time.
