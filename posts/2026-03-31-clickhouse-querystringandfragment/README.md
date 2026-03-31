# How to Use queryStringAndFragment() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, URL Function, Analytics, Web Analytics, Query String

Description: Learn how queryStringAndFragment() extracts everything after the path in a URL including query string and fragment, making it easy to preserve or strip trailing URL components.

---

`queryStringAndFragment()` is a ClickHouse URL function that returns the combined query string and fragment portion of a URL - everything after the path, including both the `?...` and `#...` parts, but without a leading `?`. This is distinct from `queryString()` (which stops at `#`) and `fragment()` (which starts after `#`). When you need to capture the full trailing context of a URL in a single call, `queryStringAndFragment()` is the right tool.

Given `https://example.com/search?q=hello&lang=en#results`, the function returns `q=hello&lang=en#results`.

## Basic Usage

```sql
-- Compare queryString(), fragment(), and queryStringAndFragment()
SELECT
    url,
    queryString(url)            AS qs_only,
    fragment(url)               AS frag_only,
    queryStringAndFragment(url) AS qs_and_frag
FROM (
    SELECT arrayJoin([
        'https://example.com/search?q=clickhouse&lang=en#top',
        'https://shop.io/cart?item=42&qty=3',
        'https://docs.io/guide#installation',
        'https://app.io/dash?tab=home#section2',
        'https://example.com/no-qs-no-frag'
    ]) AS url
);
```

```text
url                                              qs_only              frag_only     qs_and_frag
https://example.com/search?q=clickhouse&lang=en#top  q=clickhouse&lang=en  top     q=clickhouse&lang=en#top
https://shop.io/cart?item=42&qty=3               item=42&qty=3                      item=42&qty=3
https://docs.io/guide#installation                                    installation  #installation
https://app.io/dash?tab=home#section2            tab=home             section2      tab=home#section2
https://example.com/no-qs-no-frag
```

Note: when a URL has only a fragment (no query string), `queryStringAndFragment()` includes the `#` prefix.

## Stripping Trailing Components to Get a Canonical Base URL

```sql
-- Build the canonical URL by removing qs and fragment
SELECT
    url,
    -- Reconstruct base URL = protocol + domain + path
    concat(
        protocol(url), '://',
        domain(url),
        path(url)
    )  AS canonical_url,
    queryStringAndFragment(url) AS trailing
FROM page_views
WHERE event_date = yesterday()
  AND queryStringAndFragment(url) != ''
LIMIT 20;
```

## Counting Unique Trailing Variants Per Path

```sql
-- How many distinct query+fragment combinations does each path have?
SELECT
    path(url)                            AS route,
    uniq(queryStringAndFragment(url))    AS variant_count,
    count()                              AS total_hits
FROM page_views
WHERE event_date = yesterday()
GROUP BY route
ORDER BY variant_count DESC
LIMIT 30;
```

## Detecting Duplicate Content (Same Path, Different Trailing)

```sql
-- Pages that serve the same content under many trailing variants
SELECT
    path(url) AS route,
    groupArray(DISTINCT queryStringAndFragment(url)) AS variants,
    count()   AS hits
FROM page_views
WHERE event_date BETWEEN today() - 7 AND today()
GROUP BY route
HAVING uniq(queryStringAndFragment(url)) > 10
ORDER BY hits DESC
LIMIT 20;
```

## Preserving Full Context for Redirect Logging

```sql
-- Log which trailing context was present before a redirect
SELECT
    ts,
    client_ip,
    path(original_url)                     AS from_path,
    queryStringAndFragment(original_url)   AS from_trailing,
    redirect_target
FROM redirect_logs
WHERE toDate(ts) = yesterday()
  AND queryStringAndFragment(original_url) != ''
ORDER BY ts DESC
LIMIT 100;
```

## Comparing Cache Hit Rates With and Without Trailing Components

```sql
-- Cache hit rates: requests with trailing context vs clean paths
SELECT
    path(url)                                                         AS route,
    countIf(queryStringAndFragment(url) = '' AND cache_hit = 1)       AS clean_cached,
    countIf(queryStringAndFragment(url) = '' AND cache_hit = 0)       AS clean_miss,
    countIf(queryStringAndFragment(url) != '' AND cache_hit = 1)      AS trailing_cached,
    countIf(queryStringAndFragment(url) != '' AND cache_hit = 0)      AS trailing_miss
FROM access_logs
WHERE toDate(ts) = yesterday()
GROUP BY route
ORDER BY (clean_miss + trailing_miss) DESC
LIMIT 30;
```

## Normalising URLs for Deduplication

```sql
-- Deduplicate page view events by canonical URL within a session
SELECT
    session_id,
    concat(domain(url), path(url)) AS canonical_url,
    min(ts)                        AS first_seen,
    max(ts)                        AS last_seen,
    count()                        AS view_count
FROM page_views
WHERE event_date = yesterday()
GROUP BY session_id, canonical_url
ORDER BY session_id, first_seen;
```

## Summary

`queryStringAndFragment()` returns the entire trailing portion of a URL as a single string, covering both query string and fragment without requiring you to call two separate functions and concatenate results. It is most useful when you want to strip or preserve the full trailing context as a unit - for example, when constructing canonical base URLs, counting content variants, or auditing redirect chains. For extracting specific parameter values, use `extractURLParameter()` on the query string portion.
