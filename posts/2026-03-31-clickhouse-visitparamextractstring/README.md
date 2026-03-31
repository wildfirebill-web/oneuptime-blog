# How to Use visitParamExtractString() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Json, Analytics, Query

Description: Learn how visitParamExtractString() extracts string values from visit parameter JSON in ClickHouse, serving as an alias for simpleJSONExtractString() for analytics use cases.

---

`visitParamExtractString` is a ClickHouse function that extracts a string field from a JSON-formatted visit parameter string. It is functionally identical to `simpleJSONExtractString` and exists as a named alias from ClickHouse's origins in web analytics, where "visit params" were stored as flat JSON strings describing user sessions. In modern ClickHouse usage it works on any flat JSON object, not just visit data.

## Basic Usage

```sql
-- Extract visit parameters from a flat JSON string
SELECT
    visitParamExtractString(
        '{"utm_source": "google", "utm_medium": "cpc", "landing": "/pricing"}',
        'utm_source'
    ) AS utm_source;
```

```text
utm_source
google
```

## Extracting Multiple Visit Parameters

```sql
-- Parse campaign tracking fields from a visit params column
SELECT
    visit_id,
    visitParamExtractString(visit_params, 'utm_source')   AS utm_source,
    visitParamExtractString(visit_params, 'utm_medium')   AS utm_medium,
    visitParamExtractString(visit_params, 'utm_campaign') AS utm_campaign,
    visitParamExtractString(visit_params, 'landing_page') AS landing_page
FROM web_visits
LIMIT 20;
```

## Aggregating by Campaign Source

```sql
-- Count sessions and conversions per UTM source
SELECT
    visitParamExtractString(visit_params, 'utm_source') AS source,
    count()                                              AS sessions,
    countIf(JSONExtractBool(visit_params, 'converted') = 1) AS conversions
FROM web_visits
WHERE visitParamExtractString(visit_params, 'utm_source') != ''
GROUP BY source
ORDER BY sessions DESC
LIMIT 20;
```

## Filtering by Extracted Value

```sql
-- Analyze only organic search visits
SELECT
    toDate(visit_time)                                   AS visit_date,
    count()                                              AS sessions,
    uniq(user_id)                                        AS unique_users
FROM web_visits
WHERE visitParamExtractString(visit_params, 'utm_medium') = 'organic'
GROUP BY visit_date
ORDER BY visit_date;
```

## Comparing visitParamExtractString with simpleJSONExtractString

The two functions are interchangeable for flat JSON strings.

```sql
-- Both return the same result
SELECT
    visitParamExtractString('{"channel": "email"}', 'channel')   AS visit_param_result,
    simpleJSONExtractString('{"channel": "email"}', 'channel')   AS simple_json_result;
```

```text
visit_param_result  simple_json_result
email               email
```

Choose `visitParamExtractString` when the context is web analytics data to make the intent clear to readers; use `simpleJSONExtractString` for general JSON payloads.

## Handling Missing or Empty Values

```sql
-- Substitute a default when the field is absent
SELECT
    visit_id,
    if(
        visitParamExtractString(visit_params, 'referrer') = '',
        'direct',
        visitParamExtractString(visit_params, 'referrer')
    ) AS referrer
FROM web_visits
LIMIT 10;
```

## Building a Channel Attribution Report

```sql
-- Group visits by channel with fallback for missing UTM data
SELECT
    if(
        visitParamExtractString(visit_params, 'utm_medium') = '',
        'direct / none',
        concat(
            visitParamExtractString(visit_params, 'utm_source'),
            ' / ',
            visitParamExtractString(visit_params, 'utm_medium')
        )
    ) AS channel,
    count()       AS sessions,
    uniq(user_id) AS unique_users
FROM web_visits
WHERE toDate(visit_time) >= today() - 30
GROUP BY channel
ORDER BY sessions DESC;
```

## Summary

`visitParamExtractString` extracts a string field from a flat JSON object using the same heuristic parser as `simpleJSONExtractString`. The two functions are aliases of each other; the `visitParam*` family exists for readability in web analytics contexts. Use it for UTM parameters, landing page tracking, and session attribute extraction from flat JSON strings. For nested JSON or complex payloads, use the full `JSONExtractString` with path arguments.
