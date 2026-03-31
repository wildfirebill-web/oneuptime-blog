# How to Analyze DNS Logs at Scale with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, DNS, Log Analysis, Security, Network

Description: Analyze DNS logs at scale in ClickHouse to detect DNS tunneling, domain generation algorithms, and anomalous query patterns.

---

DNS logs capture every domain resolution request on your network. At enterprise scale, this generates billions of records per day. ClickHouse compresses and queries this data efficiently, enabling security and network analytics that would be impractical elsewhere.

## DNS Query Log Table

```sql
CREATE TABLE dns_queries
(
    query_id UUID DEFAULT generateUUIDv4(),
    client_ip IPv4,
    server_ip IPv4,
    query_name String,
    query_type LowCardinality(String),
    response_code UInt8,
    answer String DEFAULT '',
    response_time_ms UInt16,
    query_size UInt16,
    response_size UInt32,
    queried_at DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(queried_at)
ORDER BY (client_ip, queried_at)
TTL toDate(queried_at) + INTERVAL 90 DAY;
```

## Top Queried Domains

```sql
SELECT
    query_name,
    query_type,
    count() AS query_count,
    countDistinct(client_ip) AS unique_clients,
    countIf(response_code != 0) AS failed_queries
FROM dns_queries
WHERE queried_at >= now() - INTERVAL 1 HOUR
GROUP BY query_name, query_type
ORDER BY query_count DESC
LIMIT 30;
```

## NXDOMAIN (Non-Existent Domain) Spikes

High NXDOMAIN rates often indicate malware C2 beaconing or DGA activity.

```sql
SELECT
    client_ip,
    count() AS total_queries,
    countIf(response_code = 3) AS nxdomain_count,
    round(countIf(response_code = 3) * 100.0 / count(), 2) AS nxdomain_pct
FROM dns_queries
WHERE queried_at >= now() - INTERVAL 1 HOUR
GROUP BY client_ip
HAVING nxdomain_count > 50
ORDER BY nxdomain_count DESC
LIMIT 20;
```

## DNS Tunneling Detection - High Query Entropy

DNS tunneling uses long, high-entropy subdomains to exfiltrate data.

```sql
SELECT
    client_ip,
    query_name,
    length(query_name) AS name_length,
    response_size,
    queried_at
FROM dns_queries
WHERE queried_at >= now() - INTERVAL 1 HOUR
  AND length(query_name) > 100
  AND query_type = 'TXT'
ORDER BY length(query_name) DESC
LIMIT 30;
```

## High Volume DNS Clients

```sql
SELECT
    client_ip,
    count() AS query_count,
    countDistinct(query_name) AS unique_domains,
    countIf(response_code = 0) AS successful,
    countIf(response_code != 0) AS failed,
    round(avg(response_time_ms), 1) AS avg_latency_ms
FROM dns_queries
WHERE queried_at >= now() - INTERVAL 1 HOUR
GROUP BY client_ip
ORDER BY query_count DESC
LIMIT 20;
```

## Newly Registered Domain Detection

Domains queried for the first time in the last 24 hours.

```sql
SELECT
    query_name,
    client_ip,
    count() AS queries,
    min(queried_at) AS first_query
FROM dns_queries
WHERE queried_at >= now() - INTERVAL 24 HOUR
  AND query_name NOT IN (
      SELECT DISTINCT query_name
      FROM dns_queries
      WHERE queried_at < now() - INTERVAL 24 HOUR
        AND queried_at >= now() - INTERVAL 30 DAY
  )
GROUP BY query_name, client_ip
ORDER BY queries DESC
LIMIT 30;
```

## DNS Query Type Distribution

```sql
SELECT
    query_type,
    count() AS queries,
    round(count() * 100.0 / sum(count()) OVER (), 2) AS pct
FROM dns_queries
WHERE queried_at >= now() - INTERVAL 1 HOUR
GROUP BY query_type
ORDER BY queries DESC;
```

## Resolver Response Time Percentiles

```sql
SELECT
    server_ip,
    count() AS queries,
    round(quantile(0.50)(response_time_ms), 1) AS p50_ms,
    round(quantile(0.95)(response_time_ms), 1) AS p95_ms,
    round(quantile(0.99)(response_time_ms), 1) AS p99_ms
FROM dns_queries
WHERE queried_at >= now() - INTERVAL 1 HOUR
GROUP BY server_ip
ORDER BY p95_ms DESC;
```

## Summary

ClickHouse handles DNS log analysis at scale through fast aggregation on partitioned, columnar data. NXDOMAIN spike detection, DNS tunneling identification via query length analysis, and first-seen domain queries are all practical at billions of records per day. Combined with threat intelligence correlation, DNS log analysis in ClickHouse provides a powerful layer of network security visibility.
