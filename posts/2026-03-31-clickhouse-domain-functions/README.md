# How to Use domain() and domainWithoutWWW() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Url, Analytics, Query

Description: Learn how domain() and domainWithoutWWW() extract the hostname from URL strings in ClickHouse, enabling domain-level traffic analysis and referrer grouping.

---

`domain` and `domainWithoutWWW` are URL parsing functions in ClickHouse that extract the hostname component from a URL string. `domain` returns the full hostname including any subdomain, while `domainWithoutWWW` strips a leading `www.` prefix if present. Both return an empty string on invalid or non-URL input. They are commonly used to group web traffic by referring site, analyze outbound link domains, or bucket URLs by host in analytics queries.

## Basic Usage

```sql
-- Extract domain and domain without www from URL literals
SELECT
    domain('https://www.example.com/path?q=1')          AS full_domain,
    domainWithoutWWW('https://www.example.com/path?q=1') AS domain_no_www;
```

```text
full_domain      domain_no_www
www.example.com  example.com
```

## Handling Subdomains

`domainWithoutWWW` only removes a leading `www.` prefix; other subdomains are preserved.

```sql
SELECT
    domain('https://blog.example.com/post/1')          AS full_domain,
    domainWithoutWWW('https://blog.example.com/post/1') AS domain_no_www;
```

```text
full_domain        domain_no_www
blog.example.com   blog.example.com
```

## Extracting Domains from a Table Column

```sql
-- Extract the referring domain from a page views table
SELECT
    page_view_id,
    referrer_url,
    domain(referrer_url)          AS referrer_domain,
    domainWithoutWWW(referrer_url) AS referrer_root_domain
FROM page_views
WHERE referrer_url != ''
LIMIT 20;
```

## Grouping Traffic by Referrer Domain

```sql
-- Count sessions per referring domain over the last 30 days
SELECT
    domainWithoutWWW(referrer_url) AS referrer_domain,
    count()                         AS sessions,
    uniq(user_id)                   AS unique_users
FROM web_sessions
WHERE
    session_start >= now() - INTERVAL 30 DAY
    AND referrer_url != ''
GROUP BY referrer_domain
ORDER BY sessions DESC
LIMIT 20;
```

## Filtering Traffic from a Specific Domain

```sql
-- Find all visits referred by google.com or its subdomains
SELECT
    session_id,
    session_start,
    landing_page
FROM web_sessions
WHERE domainWithoutWWW(referrer_url) = 'google.com'
ORDER BY session_start DESC
LIMIT 50;
```

## Handling Invalid or Empty URLs

Both functions return an empty string for non-URL input.

```sql
-- Check what happens with malformed or missing URLs
SELECT
    domain('')                  AS empty_url,
    domain('not a url')         AS invalid_url,
    domain('http://valid.com/') AS valid_url;
```

```text
empty_url  invalid_url  valid_url
(empty)    (empty)      valid.com
```

Use this to filter out rows with missing referrer data.

```sql
-- Exclude rows where the URL is invalid or missing
SELECT
    domainWithoutWWW(referrer_url) AS referrer_domain,
    count()                         AS sessions
FROM web_sessions
WHERE domain(referrer_url) != ''
GROUP BY referrer_domain
ORDER BY sessions DESC
LIMIT 20;
```

## Comparing Internal vs External Traffic

```sql
-- Split traffic by whether the referrer is from your own domain
SELECT
    CASE
        WHEN domain(referrer_url) = '' THEN 'direct'
        WHEN domainWithoutWWW(referrer_url) = 'mysite.com' THEN 'internal'
        ELSE domainWithoutWWW(referrer_url)
    END AS traffic_source,
    count()       AS sessions,
    uniq(user_id) AS unique_users
FROM web_sessions
WHERE session_start >= today() - 7
GROUP BY traffic_source
ORDER BY sessions DESC;
```

## Building a Domain Allowlist Filter

```sql
-- Only keep events from known partner domains
SELECT
    event_id,
    referrer_url,
    domainWithoutWWW(referrer_url) AS referrer_domain
FROM partner_events
WHERE domainWithoutWWW(referrer_url) IN (
    'partner-a.com',
    'partner-b.com',
    'partner-c.com'
)
ORDER BY event_time DESC
LIMIT 50;
```

## Summary

`domain` returns the full hostname from a URL including subdomains, and `domainWithoutWWW` strips a leading `www.` prefix for normalized domain grouping. Both return empty strings for non-URL input, so filter on `domain(col) != ''` to exclude invalid rows. Use `domainWithoutWWW` as the grouping key when you want `www.example.com` and `example.com` to count as the same referrer domain in traffic analysis.
