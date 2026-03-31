# How to Use extractURLParameters() and extractURLParameterNames() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, URL Function, Analytics, Web Analytics, Query String

Description: Learn how extractURLParameters() returns all query string key=value pairs and extractURLParameterNames() returns just the keys from a URL in ClickHouse for bulk parameter analysis.

---

While `extractURLParameter()` retrieves one named parameter, ClickHouse provides two functions for bulk extraction. `extractURLParameters(url)` returns an `Array(String)` of all `key=value` pairs from the query string. `extractURLParameterNames(url)` returns an `Array(String)` containing just the parameter names (keys). Both functions make it easy to perform set operations, enumerate unknown parameters, or audit what parameters are present across thousands of URLs.

## Basic Usage

```sql
-- Show all parameters and their names for sample URLs
SELECT
    url,
    extractURLParameters(url)     AS all_params,
    extractURLParameterNames(url) AS param_names
FROM (
    SELECT arrayJoin([
        'https://example.com/?utm_source=google&utm_medium=cpc&utm_campaign=sale',
        'https://shop.io/cart?item=42&qty=3&coupon=SAVE10',
        'https://api.io/data?format=json&limit=100&offset=0',
        'https://example.com/no-params'
    ]) AS url
);
```

```text
url                                           all_params                                         param_names
https://example.com/?utm_source=google...     ['utm_source=google','utm_medium=cpc',...]         ['utm_source','utm_medium','utm_campaign']
https://shop.io/cart?item=42&qty=3&...        ['item=42','qty=3','coupon=SAVE10']                 ['item','qty','coupon']
https://api.io/data?format=json&limit=100...  ['format=json','limit=100','offset=0']             ['format','limit','offset']
https://example.com/no-params                 []                                                 []
```

## Counting Parameter Usage Across All URLs

```sql
-- Which query string parameters appear most often in your logs?
SELECT
    param_name,
    count() AS occurrences,
    uniq(url) AS unique_urls
FROM (
    SELECT
        url,
        arrayJoin(extractURLParameterNames(url)) AS param_name
    FROM page_views
    WHERE event_date = yesterday()
      AND queryString(url) != ''
)
GROUP BY param_name
ORDER BY occurrences DESC
LIMIT 30;
```

## Detecting Unexpected Parameters

```sql
-- Find URLs that contain parameters outside the known allowed set
SELECT
    url,
    arrayFilter(
        p -> NOT has(['utm_source','utm_medium','utm_campaign','ref','lang'], p),
        extractURLParameterNames(url)
    ) AS unknown_params
FROM page_views
WHERE event_date = yesterday()
  AND length(arrayFilter(
        p -> NOT has(['utm_source','utm_medium','utm_campaign','ref','lang'], p),
        extractURLParameterNames(url)
    )) > 0
LIMIT 30;
```

## Checking UTM Parameter Completeness

```sql
-- Sessions that have some UTM params but not all three (incomplete tagging)
SELECT
    url,
    extractURLParameterNames(url) AS present_params,
    hasAll(extractURLParameterNames(url), ['utm_source','utm_medium','utm_campaign']) AS fully_tagged
FROM page_views
WHERE event_date = yesterday()
  AND hasAny(extractURLParameterNames(url), ['utm_source','utm_medium','utm_campaign'])
  AND NOT hasAll(extractURLParameterNames(url), ['utm_source','utm_medium','utm_campaign'])
LIMIT 30;
```

## Parameter Count Distribution

```sql
-- Distribution of query string parameter counts per request
SELECT
    length(extractURLParameterNames(url)) AS param_count,
    count()                               AS requests
FROM page_views
WHERE event_date = yesterday()
GROUP BY param_count
ORDER BY param_count;
```

## Exploding All Parameters for Full ELT Pipeline

```sql
-- Flatten all key=value pairs into one row each for downstream processing
SELECT
    toDate(ts)    AS log_date,
    url,
    splitByChar('=', kv)[1] AS param_key,
    splitByChar('=', kv)[2] AS param_value
FROM (
    SELECT
        ts,
        url,
        arrayJoin(extractURLParameters(url)) AS kv
    FROM access_logs
    WHERE toDate(ts) = yesterday()
      AND queryString(url) != ''
)
ORDER BY log_date, url, param_key;
```

## Finding Co-occurring Parameter Combinations

```sql
-- Top 10 most common combinations of parameter names (sorted for normalisation)
SELECT
    arrayStringConcat(arraySort(extractURLParameterNames(url)), '+') AS param_combo,
    count() AS hits
FROM page_views
WHERE event_date = yesterday()
  AND length(extractURLParameterNames(url)) > 0
GROUP BY param_combo
ORDER BY hits DESC
LIMIT 20;
```

## Auditing for Sensitive Parameter Leakage

```sql
-- Find any URL where a sensitive key name appears
SELECT
    ts,
    client_ip,
    url
FROM access_logs
WHERE toDate(ts) = today()
  AND hasAny(
        extractURLParameterNames(url),
        ['password', 'token', 'secret', 'api_key', 'access_token', 'auth']
      )
ORDER BY ts DESC
LIMIT 100;
```

## Summary

`extractURLParameters()` and `extractURLParameterNames()` complement `extractURLParameter()` by operating on the full parameter set at once. Use `extractURLParameterNames()` when you need to check for parameter presence, count parameters, or detect unexpected keys. Use `extractURLParameters()` when you need all `key=value` strings for further parsing or ELT ingestion. Both return arrays, making them natural inputs to `arrayJoin`, `hasAny`, `hasAll`, and other array functions.
