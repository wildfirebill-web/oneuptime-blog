# How to Use replaceRegexpOne() and replaceRegexpAll() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, String Function, Regex, Data Masking, Log Parsing

Description: Learn how replaceRegexpOne() and replaceRegexpAll() work in ClickHouse, including backreference syntax for URL normalization, log parsing, and PII masking.

---

When a fixed-string substitution is not precise enough, ClickHouse offers two regex-based replacement functions: `replaceRegexpOne()` and `replaceRegexpAll()`. Both accept a regular expression as the search pattern and support backreferences in the replacement string, giving you the power to restructure matched content rather than just overwrite it.

## Function Signatures

```text
replaceRegexpOne(haystack, pattern, replacement)
replaceRegexpAll(haystack, pattern, replacement)
```

`replaceRegexpOne` replaces the first match of `pattern` in `haystack` with `replacement` and then stops.

`replaceRegexpAll` replaces every non-overlapping match from left to right.

Both functions use the RE2 regular expression syntax. Backreferences in the replacement string are written as `\1`, `\2`, and so on, corresponding to the capture groups in the pattern. To include a literal backslash in the replacement, write `\\`.

## Basic Usage

The following examples show the difference in behavior when the pattern matches more than once:

```sql
-- replaceRegexpOne stops after the first match
SELECT replaceRegexpOne('2024-01-15 2024-02-20', '\\d{4}', 'YYYY') AS result;
-- result: 'YYYY-01-15 2024-02-20'

-- replaceRegexpAll replaces every match
SELECT replaceRegexpAll('2024-01-15 2024-02-20', '\\d{4}', 'YYYY') AS result;
-- result: 'YYYY-01-15 YYYY-02-20'
```

Note that in ClickHouse string literals, backslashes must be doubled: `\\d` in the SQL string represents the regex token `\d`.

## Using Backreferences to Restructure Matched Content

Backreferences let you capture parts of the matched text and reuse them in the replacement. The replacement string `\1` refers to the text captured by the first parenthesized group.

The example below reformats ISO dates from `YYYY-MM-DD` to `DD/MM/YYYY`:

```sql
SELECT replaceRegexpOne(
    '2024-03-15',
    '(\\d{4})-(\\d{2})-(\\d{2})',
    '\\3/\\2/\\1'
) AS reformatted;
-- result: '15/03/2024'
```

The three capture groups `(\d{4})`, `(\d{2})`, `(\d{2})` bind to the year, month, and day. The replacement `\3/\2/\1` reassembles them in day/month/year order.

## URL Normalization

Log data frequently contains URLs with inconsistent trailing slashes, query strings, or mixed-case schemes. `replaceRegexpOne()` is well suited to normalizing these before aggregation.

Strip a trailing slash from a path:

```sql
SELECT replaceRegexpOne(url_path, '/$', '') AS clean_path
FROM page_views
LIMIT 10;
```

Remove the query string entirely, keeping only the path:

```sql
SELECT replaceRegexpOne(url, '\\?.*$', '') AS path_only
FROM page_views
LIMIT 10;
```

Normalize the scheme to lowercase:

```sql
SELECT replaceRegexpOne(url, '^HTTP://', 'https://') AS normalized_url
FROM page_views
WHERE url LIKE 'HTTP://%'
LIMIT 10;
```

For URLs that may contain multiple query parameters, combine normalization steps:

```sql
SELECT
    url,
    replaceRegexpOne(
        replaceRegexpOne(lower(url), '\\?.*$', ''),
        '/$', ''
    ) AS canonical_url
FROM page_views
LIMIT 10;
```

## Log Parsing

Structured log lines often embed key-value pairs or follow a known template. `replaceRegexpAll()` with backreferences lets you extract or reformat those embedded values inline.

The following query extracts the HTTP status code from a combined log format line and replaces the surrounding text with just the code:

```sql
SELECT replaceRegexpOne(
    '127.0.0.1 - - [15/Mar/2024:10:22:01 +0000] "GET /api/health HTTP/1.1" 200 1024',
    '.* "(\\w+ [^ ]+) HTTP/[^ ]+" (\\d{3}) .*',
    '\\1 -> \\2'
) AS parsed;
-- result: 'GET /api/health -> 200'
```

To anonymize IP addresses in bulk log data, replace the last octet with a placeholder:

```sql
SELECT replaceRegexpAll(
    log_line,
    '(\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.)\\d{1,3}',
    '\\1x'
) AS anonymized_log
FROM access_logs
LIMIT 20;
```

## Data Masking for PII

Before writing sensitive data to an analytics store, you can mask personal information at the SQL layer without a separate pipeline step.

Mask all email addresses in a free-text column, preserving only the domain:

```sql
SELECT replaceRegexpAll(
    message,
    '[a-zA-Z0-9._%+\\-]+@([a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,})',
    '[masked]@\\1'
) AS safe_message
FROM support_tickets
LIMIT 10;
```

Mask credit card numbers, keeping only the last four digits:

```sql
SELECT replaceRegexpAll(
    payment_note,
    '\\b\\d{4}[\\s\\-]?\\d{4}[\\s\\-]?\\d{4}[\\s\\-]?(\\d{4})\\b',
    'xxxx-xxxx-xxxx-\\1'
) AS masked_note
FROM payment_events
LIMIT 10;
```

## Applying to Array Columns

Combine `replaceRegexpAll()` with `arrayMap()` to apply regex replacement to every element of an array column:

```sql
SELECT
    arrayMap(
        v -> replaceRegexpAll(v, '^\\s+|\\s+$', ''),
        tag_list
    ) AS trimmed_tags
FROM events
LIMIT 5;
```

This trims leading and trailing whitespace from each element of `tag_list` using the regex `^\s+|\s+$`.

## Performance Notes

ClickHouse uses the RE2 library for regex evaluation, which provides linear-time guarantees and avoids catastrophic backtracking. However, regex compilation does carry overhead compared to literal string replacement. When a substitution can be expressed as a plain substring replacement, `replaceAll()` will be faster.

For high-cardinality columns or very large tables, consider materializing the transformed value in a dedicated column using a materialized expression:

```sql
ALTER TABLE access_logs
    ADD COLUMN ip_anon String
    MATERIALIZED replaceRegexpAll(
        ip_address,
        '(\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.)\\d{1,3}',
        '\\1x'
    );
```

## Summary

`replaceRegexpOne(haystack, pattern, replacement)` replaces the first regex match and stops; `replaceRegexpAll(haystack, pattern, replacement)` replaces every match. Both support RE2 syntax and backreferences (`\1`, `\2`) in the replacement string, allowing you to restructure matched content rather than simply overwrite it. Use them for URL normalization, log transformation, and PII masking when fixed-string replacement is too coarse. For simple literal substitutions, prefer `replace()` or `replaceAll()` to avoid unnecessary regex overhead.
