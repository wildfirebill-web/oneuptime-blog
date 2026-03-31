# How to Use URLHash() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Hash Function, URL, Web Analytics, Routing

Description: Learn how to use URLHash() in ClickHouse to hash URL path prefixes for grouping, routing, and web analytics based on URL hierarchy.

---

`URLHash(url, n)` is a specialized hash function that extracts and hashes the first `n` elements of the URL path hierarchy. This makes it ideal for grouping URLs by domain, path prefix, or any level of the URL hierarchy. Unlike hashing the full URL string, `URLHash` understands URL structure and lets you hash at a specific depth in the path.

## Basic Usage

```sql
-- Hash the first element (scheme + domain)
SELECT URLHash('https://example.com/path/to/page', 1) AS domain_hash;

-- Hash the first two path elements
SELECT URLHash('https://example.com/api/v1/users', 2) AS api_prefix_hash;

-- Hash the full URL (or up to a very large depth)
SELECT URLHash('https://example.com/path/to/page', 5) AS full_url_hash;
```

## Understanding URL Depth

The `n` parameter controls how many levels of the URL hierarchy are included in the hash.

```sql
-- See what each depth level covers
SELECT
    'https://example.com/blog/post/123' AS url,
    URLHash('https://example.com/blog/post/123', 1) AS depth_1_domain,
    URLHash('https://example.com/blog/post/123', 2) AS depth_2_blog,
    URLHash('https://example.com/blog/post/123', 3) AS depth_3_post,
    URLHash('https://example.com/blog/post/123', 4) AS depth_4_full;
```

## Grouping by URL Domain

Group all traffic to the same domain, regardless of path.

```sql
-- Count page views grouped by domain
SELECT
    URLHash(page_url, 1) AS domain_hash,
    count()              AS page_views
FROM web_traffic
GROUP BY domain_hash
ORDER BY page_views DESC
LIMIT 20;
```

## Grouping by URL Path Prefix

Group URLs that share a common path prefix (such as all URLs under `/api/v1/`).

```sql
-- Count requests per API section (first 2 path levels)
SELECT
    URLHash(request_url, 2) AS section_hash,
    count()                 AS request_count
FROM api_logs
GROUP BY section_hash
ORDER BY request_count DESC
LIMIT 20;
```

## URL-Based Routing

Use `URLHash` to route requests to backend handlers based on the URL prefix.

```sql
-- Route URLs to one of 8 handlers based on the first path element
SELECT
    request_id,
    request_url,
    URLHash(request_url, 2) % 8 AS handler_id
FROM incoming_requests
LIMIT 20;
```

## Consistent Caching by URL Prefix

Assign cache servers based on URL prefix to maximize cache locality.

```sql
-- Assign URLs to cache servers based on the domain+path level
SELECT
    page_url,
    URLHash(page_url, 2) % 4 AS cache_server_id
FROM cached_pages
LIMIT 20;

-- Verify even distribution across cache servers
SELECT
    URLHash(page_url, 2) % 4 AS cache_server_id,
    count()                   AS pages_assigned
FROM cached_pages
GROUP BY cache_server_id
ORDER BY cache_server_id;
```

## Web Analytics: Top Path Prefixes

Identify the most visited sections of a website by hashing path prefixes and then resolving the hash back using a join or conditional logic.

```sql
-- Find the most common first-level path sections
SELECT
    URLHash(page_url, 2) AS section_hash,
    count()              AS visits,
    min(page_url)        AS example_url
FROM page_views
WHERE toDate(visit_time) = today()
GROUP BY section_hash
ORDER BY visits DESC
LIMIT 10;
```

## Detecting Traffic Spikes per URL Section

```sql
-- Detect traffic spikes per URL section over time
SELECT
    toStartOfHour(request_time) AS hour,
    URLHash(url, 2)             AS section_hash,
    count()                     AS requests_per_hour
FROM http_logs
WHERE request_time >= now() - INTERVAL 24 HOUR
GROUP BY hour, section_hash
ORDER BY hour, requests_per_hour DESC;
```

## Summary

`URLHash(url, n)` hashes the first `n` elements of a URL's path hierarchy, returning a UInt64. This makes it far more useful than hashing a full URL string when you want to group or route traffic by URL domain or path prefix. Use depth 1 to group by domain, depth 2 to group by the first path segment, and so on. It is particularly valuable in web analytics, API routing, and cache distribution scenarios.
