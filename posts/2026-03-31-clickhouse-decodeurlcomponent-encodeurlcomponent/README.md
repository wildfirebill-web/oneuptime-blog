# How to Use decodeURLComponent() and encodeURLComponent() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, URL Function, Analytics, Encoding, Web Analytics

Description: Learn how decodeURLComponent() decodes percent-encoded URL strings and encodeURLComponent() encodes special characters in ClickHouse for safe URL construction and log normalisation.

---

`decodeURLComponent(str)` converts a percent-encoded string into a plain UTF-8 string, replacing sequences like `%20` with a space and `%C3%A9` with `e` (with accent). `encodeURLComponent(str)` does the reverse: it percent-encodes all characters that are not unreserved URL characters (letters, digits, `-`, `_`, `.`, `~`), producing a string safe for embedding in a URL query string.

These two functions are essential when logs contain encoded URLs that need to be read as plain text, or when constructing URLs programmatically from user-supplied data.

## Basic Usage

```sql
-- Decode percent-encoded strings
SELECT
    raw,
    decodeURLComponent(raw) AS decoded
FROM (
    SELECT arrayJoin([
        'hello%20world',
        'caf%C3%A9',
        'price%3A%24100',
        'search%3Fq%3Dclick%2Bhouse',
        'normal_string'
    ]) AS raw
);
```

```text
raw                       decoded
hello%20world             hello world
caf%C3%A9                 café
price%3A%24100            price:$100
search%3Fq%3Dclick%2Bhouse  search?q=click+house
normal_string             normal_string
```

```sql
-- Encode plain strings for safe URL embedding
SELECT
    plain,
    encodeURLComponent(plain) AS encoded
FROM (
    SELECT arrayJoin([
        'hello world',
        'price: $100',
        'user@example.com',
        'path/to/resource',
        'simple'
    ]) AS plain
);
```

```text
plain               encoded
hello world         hello%20world
price: $100         price%3A%20%24100
user@example.com    user%40example.com
path/to/resource    path%2Fto%2Fresource
simple              simple
```

## Decoding Search Queries in Logs

```sql
-- Readable search terms from encoded query parameters
SELECT
    decodeURLComponent(extractURLParameter(url, 'q')) AS search_term,
    count()                                           AS searches,
    uniq(visitor_id)                                  AS unique_searchers
FROM page_views
WHERE event_date = yesterday()
  AND path(url) = '/search'
  AND extractURLParameter(url, 'q') != ''
GROUP BY search_term
ORDER BY searches DESC
LIMIT 30;
```

## Normalising Encoded vs Unencoded URLs

```sql
-- Group identical pages that appear both encoded and unencoded
SELECT
    decodeURLComponent(path(url)) AS decoded_path,
    count()                       AS hits,
    uniq(visitor_id)              AS unique_visitors
FROM page_views
WHERE event_date = yesterday()
GROUP BY decoded_path
ORDER BY hits DESC
LIMIT 30;
```

## Constructing Safe Redirect URLs

```sql
-- Build a properly encoded redirect URL for each session
SELECT
    session_id,
    concat(
        'https://example.com/login?next=',
        encodeURLComponent(intended_url)
    ) AS redirect_url
FROM pending_sessions
WHERE created_at >= now() - INTERVAL 1 HOUR
LIMIT 20;
```

## Detecting Double-Encoded URLs

```sql
-- Find URLs where decoding once still leaves percent signs (double-encoded)
SELECT
    url,
    decodeURLComponent(url)                          AS once_decoded,
    decodeURLComponent(decodeURLComponent(url))      AS twice_decoded,
    once_decoded != twice_decoded                    AS was_double_encoded
FROM access_logs
WHERE toDate(ts) = yesterday()
  AND url LIKE '%25%'
LIMIT 30;
```

## Round-Trip Verification

```sql
-- Verify that encode then decode returns the original string
SELECT
    original,
    decodeURLComponent(encodeURLComponent(original)) AS round_tripped,
    original = decodeURLComponent(encodeURLComponent(original)) AS is_lossless
FROM (
    SELECT arrayJoin([
        'hello world',
        'Special: <>&"',
        'UTF-8: naïve résumé',
        'Symbols: #?=&+'
    ]) AS original
);
```

```text
original               round_tripped          is_lossless
hello world            hello world            1
Special: <>&"          Special: <>&"          1
UTF-8: naïve résumé    UTF-8: naïve résumé    1
Symbols: #?=&+         Symbols: #?=&+         1
```

## Cleaning Encoded User-Agent Strings

```sql
-- Decode user-agent strings that arrive percent-encoded
SELECT
    decodeURLComponent(user_agent) AS clean_ua,
    count()                        AS requests
FROM access_logs
WHERE toDate(ts) = yesterday()
  AND user_agent LIKE '%25%'
GROUP BY clean_ua
ORDER BY requests DESC
LIMIT 20;
```

## Summary

`decodeURLComponent()` converts percent-encoded URL strings to human-readable UTF-8, making encoded logs readable and grouping together what should be treated as the same value. `encodeURLComponent()` prepares plain text strings for safe use in URL query strings, ensuring special characters do not break URL parsing. Use them together for round-trip transformations, and use `decodeURLComponent()` on extracted parameter values whenever user-supplied text may have been encoded before storage.
