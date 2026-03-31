# How to Use queryString() and fragment() URL Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, URL Function, Analytics, Web Analytics, Query String

Description: Learn how queryString() extracts the URL query string and fragment() extracts the hash fragment in ClickHouse for parameter-level traffic analysis and campaign tracking.

---

ClickHouse provides two functions for isolating specific parts of a URL beyond the path. `queryString()` returns everything after the `?` and before the `#`, without the leading `?`. `fragment()` returns everything after `#`, without the leading `#`. If the respective delimiter is absent, both functions return an empty string.

These functions are commonly used in web analytics to extract UTM campaign parameters, analyse search queries passed as URL parameters, and understand client-side routing state encoded in fragments.

## Basic Usage

```sql
-- Show the query string and fragment for various URLs
SELECT
    url,
    queryString(url)  AS qs,
    fragment(url)     AS frag
FROM (
    SELECT arrayJoin([
        'https://example.com/search?q=clickhouse&lang=en#top',
        'https://shop.io/cart?item=42&qty=3',
        'https://docs.example.com/guide#installation',
        'https://example.com/page',
        'https://app.io/dashboard?tab=overview#section2'
    ]) AS url
);
```

```text
url                                               qs                       frag
https://example.com/search?q=clickhouse&lang=en#top  q=clickhouse&lang=en  top
https://shop.io/cart?item=42&qty=3                item=42&qty=3
https://docs.example.com/guide#installation                                installation
https://example.com/page
https://app.io/dashboard?tab=overview#section2    tab=overview             section2
```

## Extracting Campaign Parameters from Query Strings

```sql
-- Parse UTM parameters from marketing URLs
SELECT
    extractURLParameter(url, 'utm_source')   AS source,
    extractURLParameter(url, 'utm_medium')   AS medium,
    extractURLParameter(url, 'utm_campaign') AS campaign,
    count()                                  AS sessions,
    uniq(visitor_id)                         AS unique_visitors
FROM page_views
WHERE event_date BETWEEN today() - 30 AND today()
  AND queryString(url) LIKE '%utm_%'
GROUP BY source, medium, campaign
ORDER BY sessions DESC
LIMIT 30;
```

## Finding the Most Common Query Strings

```sql
-- Top 20 raw query strings (helps spot parameter pollution)
SELECT
    queryString(url) AS qs,
    count()          AS hits
FROM page_views
WHERE event_date = yesterday()
  AND queryString(url) != ''
GROUP BY qs
ORDER BY hits DESC
LIMIT 20;
```

## Fragment-Based Routing Analysis

Single-page applications often use hash routing. The fragment tells you which view the user landed on.

```sql
-- Distribution of client-side routes via fragment
SELECT
    fragment(url) AS client_route,
    count()       AS page_loads,
    uniq(visitor_id) AS unique_users
FROM page_views
WHERE event_date = yesterday()
  AND fragment(url) != ''
GROUP BY client_route
ORDER BY page_loads DESC
LIMIT 30;
```

## Detecting URLs With Both Query String and Fragment

```sql
-- Find requests that carry both a query string and a fragment
SELECT
    url,
    queryString(url) AS qs,
    fragment(url)    AS frag,
    count()          AS hits
FROM page_views
WHERE event_date = yesterday()
  AND queryString(url) != ''
  AND fragment(url) != ''
GROUP BY url, qs, frag
ORDER BY hits DESC
LIMIT 20;
```

## Checking for Sensitive Parameters in Query Strings

```sql
-- Detect potential credential leakage in query strings
SELECT
    ts,
    client_ip,
    url,
    queryString(url) AS qs
FROM access_logs
WHERE toDate(ts) = today()
  AND (
      queryString(url) ILIKE '%password%'
      OR queryString(url) ILIKE '%token%'
      OR queryString(url) ILIKE '%secret%'
      OR queryString(url) ILIKE '%api_key%'
  )
ORDER BY ts DESC
LIMIT 100;
```

## Query String Length Distribution

```sql
-- Analyse query string lengths to detect unusually long requests
SELECT
    length(queryString(url))  AS qs_length,
    count()                   AS requests
FROM access_logs
WHERE toDate(ts) = yesterday()
  AND queryString(url) != ''
GROUP BY qs_length
ORDER BY qs_length
LIMIT 50;
```

## Stripping Query Strings for Cache Key Analysis

```sql
-- Compare hit rates for paths with and without query strings
SELECT
    path(url)                                         AS route,
    countIf(queryString(url) = '')                    AS hits_no_qs,
    countIf(queryString(url) != '')                   AS hits_with_qs,
    countIf(queryString(url) != '') * 100.0 / count() AS pct_with_qs
FROM access_logs
WHERE toDate(ts) = yesterday()
GROUP BY route
ORDER BY hits_with_qs DESC
LIMIT 30;
```

## Summary

`queryString()` and `fragment()` give direct access to the two trailing components of a URL. Use `queryString()` to parse campaign tracking parameters, detect parameter pollution, or audit sensitive data leakage in URLs. Use `fragment()` to analyse single-page application routing and anchor navigation patterns. For extracting specific named parameters from the query string, combine these functions with `extractURLParameter()`.
