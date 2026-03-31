# How to Use path() and pathFull() URL Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, URL Function, Analytics, Web Analytics, PATH

Description: Learn how path() returns the URL path without query string or fragment, while pathFull() includes them, enabling precise route-level traffic analysis in ClickHouse.

---

ClickHouse provides two complementary URL path functions. `path()` returns the path component of a URL - everything from the first `/` after the host up to (but not including) the `?` query string separator. `pathFull()` returns the same path but also includes the query string and fragment. Both functions strip the scheme and authority (host + port), giving you the server-relative portion of the URL.

Understanding the difference: given `https://example.com/search?q=hello#results`, `path()` returns `/search` while `pathFull()` returns `/search?q=hello#results`.

## Basic Usage

```sql
-- Compare path() and pathFull() on the same URLs
SELECT
    url,
    path(url)     AS clean_path,
    pathFull(url) AS full_path
FROM (
    SELECT arrayJoin([
        'https://example.com/products/shoes?color=red&size=10#reviews',
        'https://api.example.com/v2/users/42',
        'https://shop.io/checkout?step=payment',
        'https://example.com/',
        'https://example.com'
    ]) AS url
);
```

```text
url                                                        clean_path          full_path
https://example.com/products/shoes?color=red&size=10#...  /products/shoes     /products/shoes?color=red&size=10#reviews
https://api.example.com/v2/users/42                        /v2/users/42        /v2/users/42
https://shop.io/checkout?step=payment                      /checkout           /checkout?step=payment
https://example.com/                                       /                   /
https://example.com
```

## Top Pages by Path

```sql
-- Most visited clean paths (ignoring query string variations)
SELECT
    path(url)    AS page_path,
    count()      AS page_views,
    uniq(visitor_id) AS unique_visitors
FROM page_views
WHERE event_date = yesterday()
  AND path(url) != ''
GROUP BY page_path
ORDER BY page_views DESC
LIMIT 25;
```

## Comparing Path vs Full Path Hit Counts

```sql
-- See how query string variations fragment a single logical page
SELECT
    path(url)     AS clean_path,
    pathFull(url) AS exact_path,
    count()       AS hits
FROM page_views
WHERE event_date = yesterday()
  AND path(url) = '/products'
GROUP BY clean_path, exact_path
ORDER BY hits DESC
LIMIT 20;
```

## API Endpoint Traffic Analysis

```sql
-- Count calls per API route, stripping path parameters
SELECT
    -- Replace numeric path segments to normalise routes
    replaceRegexpAll(path(url), '/[0-9]+', '/:id') AS route,
    count()                                         AS calls,
    avg(response_time_ms)                           AS avg_ms,
    quantile(0.95)(response_time_ms)                AS p95_ms
FROM api_logs
WHERE toDate(ts) = yesterday()
GROUP BY route
ORDER BY calls DESC
LIMIT 30;
```

## Extracting the First Path Segment

```sql
-- Group traffic by top-level section (first path segment)
SELECT
    splitByChar('/', path(url))[2]  AS section,
    count()                         AS hits
FROM page_views
WHERE event_date = yesterday()
  AND path(url) != '/'
GROUP BY section
ORDER BY hits DESC
LIMIT 20;
```

## Detecting Deep vs Shallow URLs

```sql
-- Distribution of URL depth (number of path segments)
SELECT
    length(splitByChar('/', path(url))) - 1  AS depth,
    count()                                  AS urls
FROM page_views
WHERE event_date = yesterday()
  AND path(url) != ''
GROUP BY depth
ORDER BY depth;
```

## Finding 404-Prone Paths

```sql
-- Paths that most frequently return 404 errors
SELECT
    path(url)   AS page_path,
    count()     AS error_count,
    uniq(client_ip) AS unique_clients
FROM access_logs
WHERE toDate(ts) >= today() - 7
  AND status_code = 404
GROUP BY page_path
ORDER BY error_count DESC
LIMIT 20;
```

## Using pathFull() to Preserve Query Context

```sql
-- Log exact URLs for debugging, but group by clean path for metrics
SELECT
    path(url)                          AS route,
    count()                            AS total_hits,
    -- Sample one exact pathFull for debugging
    anyLast(pathFull(url))             AS example_full_path,
    avg(response_time_ms)              AS avg_ms
FROM api_logs
WHERE toDate(ts) = today()
GROUP BY route
ORDER BY total_hits DESC
LIMIT 30;
```

## Path Prefix Filtering

```sql
-- All traffic under the /blog/ section
SELECT
    pathFull(url) AS full_path,
    count()       AS views,
    uniq(visitor_id) AS unique_readers
FROM page_views
WHERE event_date BETWEEN today() - 30 AND today()
  AND startsWith(path(url), '/blog/')
GROUP BY full_path
ORDER BY views DESC
LIMIT 30;
```

## Summary

`path()` gives you the clean, query-string-free path ideal for grouping page views and API routes without fragmentation from parameter variations. `pathFull()` preserves the complete server-relative URL including query string and fragment, useful when you need the exact request string for debugging or cache-key analysis. Combine both in the same query - use `path()` for aggregation keys and `pathFull()` for diagnostic sample values.
