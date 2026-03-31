# How to Build a Referrer Analysis System with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Web Analytics, Referrer, Traffic Source, URL, Analytics

Description: Build a referrer analysis system in ClickHouse to track traffic sources, parse URLs into domains and UTM parameters, and measure conversion by channel.

---

Understanding where your traffic comes from is essential for marketing decisions. ClickHouse provides built-in URL parsing functions that make referrer analysis straightforward and fast at scale.

## Schema Design

```sql
CREATE TABLE page_events (
    session_id  String,
    user_id     UInt64,
    path        LowCardinality(String),
    referrer    String,
    utm_source  LowCardinality(String),
    utm_medium  LowCardinality(String),
    utm_campaign String,
    created_at  DateTime DEFAULT now()
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, session_id);
```

## Parsing Referrer URLs

ClickHouse has built-in URL functions for parsing referrers:

```sql
SELECT
    referrer,
    domain(referrer) AS referrer_domain,
    protocol(referrer) AS referrer_protocol
FROM page_events
WHERE referrer != ''
LIMIT 10;
```

Extract top-level domain:

```sql
SELECT
    domainWithoutWWW(referrer) AS clean_domain,
    count() AS visits
FROM page_events
WHERE referrer != ''
  AND created_at >= today() - 30
GROUP BY clean_domain
ORDER BY visits DESC
LIMIT 20;
```

## Classifying Traffic Sources

Categorize referrers into channels:

```sql
SELECT
    session_id,
    referrer,
    multiIf(
        utm_medium = 'email',                                  'Email',
        utm_medium IN ('cpc', 'ppc', 'paid'),                 'Paid Search',
        utm_medium = 'social',                                 'Social',
        referrer = '' AND utm_source = '',                     'Direct',
        domainWithoutWWW(referrer) IN
            ('google.com', 'bing.com', 'yahoo.com', 'duckduckgo.com'), 'Organic Search',
        domainWithoutWWW(referrer) IN
            ('facebook.com', 'twitter.com', 'linkedin.com',
             'instagram.com', 'tiktok.com'),                   'Social',
        referrer != '',                                        'Referral',
        'Direct'
    ) AS channel
FROM page_events
WHERE created_at >= today() - 30;
```

## Sessions by Channel

```sql
SELECT
    multiIf(
        utm_medium = 'email',   'Email',
        referrer = '',          'Direct',
        domainWithoutWWW(referrer) IN ('google.com','bing.com','yahoo.com'), 'Organic',
        'Referral'
    ) AS channel,
    uniq(session_id) AS sessions,
    uniq(user_id) AS users
FROM page_events
WHERE created_at >= today() - 30
GROUP BY channel
ORDER BY sessions DESC;
```

## UTM Campaign Analysis

```sql
SELECT
    utm_source,
    utm_medium,
    utm_campaign,
    uniq(session_id) AS sessions,
    uniq(user_id) AS users
FROM page_events
WHERE utm_source != ''
  AND created_at >= today() - 30
GROUP BY utm_source, utm_medium, utm_campaign
ORDER BY sessions DESC
LIMIT 20;
```

## Materialized View for Daily Referrer Stats

```sql
CREATE TABLE referrer_daily (
    date           Date,
    referrer_domain LowCardinality(String),
    channel        LowCardinality(String),
    sessions       UInt64,
    users          UInt64
) ENGINE = SummingMergeTree((sessions, users))
ORDER BY (date, referrer_domain, channel);

CREATE MATERIALIZED VIEW referrer_daily_mv
TO referrer_daily AS
SELECT
    toDate(created_at) AS date,
    domainWithoutWWW(referrer) AS referrer_domain,
    multiIf(
        utm_medium = 'email', 'Email',
        referrer = '',        'Direct',
        domainWithoutWWW(referrer) IN ('google.com','bing.com'), 'Organic',
        'Referral'
    ) AS channel,
    uniq(session_id) AS sessions,
    uniq(user_id) AS users
FROM page_events
GROUP BY date, referrer_domain, channel;
```

## Query the Materialized View

```sql
SELECT
    referrer_domain,
    sum(sessions) AS total_sessions
FROM referrer_daily
WHERE date >= today() - 30
GROUP BY referrer_domain
ORDER BY total_sessions DESC
LIMIT 20;
```

## Summary

ClickHouse's built-in URL parsing functions (domain, domainWithoutWWW, protocol) make referrer analysis expressive and fast. Combining multiIf for channel classification with a SummingMergeTree materialized view gives you a real-time traffic source dashboard that scales to billions of events.
