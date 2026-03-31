# How to Use cutURLParameter() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, URL Function, Analytics, Web Analytics, Privacy

Description: Learn how cutURLParameter() removes a named query string parameter from a URL in ClickHouse, enabling URL canonicalisation, privacy scrubbing, and deduplication workflows.

---

`cutURLParameter(url, name)` removes all occurrences of a named query string parameter (and its value) from a URL and returns the modified URL string. The rest of the URL - scheme, host, path, other parameters, and fragment - is preserved. This function is the counterpart to `extractURLParameter()`: one reads a parameter, the other removes it.

Common use cases include stripping tracking parameters to produce canonical URLs, removing sensitive values before storing logs, and deduplicating URLs that differ only in session or click-tracking parameters.

## Basic Usage

```sql
-- Remove a single parameter from various URLs
SELECT
    url,
    cutURLParameter(url, 'utm_source') AS without_utm_source
FROM (
    SELECT arrayJoin([
        'https://example.com/?utm_source=google&utm_medium=cpc&page=1',
        'https://shop.io/product?id=42&utm_source=email&color=red',
        'https://example.com/?page=2',
        'https://example.com/no-params'
    ]) AS url
);
```

```text
url                                                   without_utm_source
https://example.com/?utm_source=google&utm_medium=cpc&page=1  https://example.com/?utm_medium=cpc&page=1
https://shop.io/product?id=42&utm_source=email&color=red      https://shop.io/product?id=42&color=red
https://example.com/?page=2                                    https://example.com/?page=2
https://example.com/no-params                                  https://example.com/no-params
```

## Stripping All UTM Parameters for Canonical URLs

Chain multiple `cutURLParameter()` calls to remove a set of tracking parameters.

```sql
-- Remove all common tracking parameters
SELECT
    url,
    cutURLParameter(
        cutURLParameter(
            cutURLParameter(
                cutURLParameter(url, 'utm_source'),
            'utm_medium'),
        'utm_campaign'),
    'utm_content') AS canonical_url
FROM page_views
WHERE event_date = yesterday()
LIMIT 20;
```

## Deduplicating Page Views by Canonical URL

```sql
-- Count unique pages after stripping tracking parameters
SELECT
    cutURLParameter(
        cutURLParameter(
            cutURLParameter(url, 'utm_source'),
        'utm_medium'),
    'utm_campaign') AS canonical_url,
    count()         AS total_views,
    uniq(visitor_id) AS unique_visitors
FROM page_views
WHERE event_date = yesterday()
GROUP BY canonical_url
ORDER BY total_views DESC
LIMIT 30;
```

## Removing Session IDs Before Log Storage

```sql
-- Sanitise URLs by removing session and token parameters before archiving
INSERT INTO cleaned_access_logs
SELECT
    ts,
    client_ip,
    status_code,
    response_time_ms,
    cutURLParameter(
        cutURLParameter(
            cutURLParameter(url, 'session_id'),
        'token'),
    'auth') AS clean_url
FROM raw_access_logs
WHERE toDate(ts) = yesterday();
```

## Finding URLs That Become Identical After Stripping Tracking

```sql
-- Detect traffic fragmented only by tracking parameters
SELECT
    cutURLParameter(
        cutURLParameter(
            cutURLParameter(url, 'utm_source'),
        'utm_medium'),
    'utm_campaign') AS canonical_url,
    groupArray(DISTINCT url) AS original_variants,
    count()                  AS total_hits
FROM page_views
WHERE event_date = yesterday()
GROUP BY canonical_url
HAVING length(original_variants) > 1
ORDER BY total_hits DESC
LIMIT 20;
```

## Removing a Parameter From a Stored URL Column

```sql
-- Update a derived table by stripping a deprecated parameter
INSERT INTO page_views_clean
SELECT
    visitor_id,
    event_date,
    cutURLParameter(url, 'old_tracker') AS url,
    ts
FROM page_views
WHERE event_date = yesterday();
```

## Comparing Pre- and Post-Cut URL Lengths

```sql
-- Measure how many bytes of tracking data are stripped per request
SELECT
    length(url)                            AS original_length,
    length(cutURLParameter(
        cutURLParameter(url, 'utm_source'),
    'utm_campaign'))                        AS cleaned_length,
    original_length - cleaned_length        AS bytes_removed
FROM page_views
WHERE event_date = yesterday()
  AND queryString(url) LIKE '%utm_%'
ORDER BY bytes_removed DESC
LIMIT 20;
```

## Summary

`cutURLParameter()` is the surgical tool for URL sanitisation in ClickHouse. By chaining calls you can strip an entire set of parameters while preserving the rest of the URL intact. Use it to canonicalise URLs for deduplication, scrub sensitive or ephemeral parameters before archiving logs, and produce clean base URLs for SEO or caching analysis. For removing all parameters at once, consider `cutQueryString()` instead.
