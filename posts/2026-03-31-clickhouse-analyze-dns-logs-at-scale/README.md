# How to Analyze DNS Logs at Scale with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, DNS, Log Analysis, Security, Network Monitoring

Description: Learn how to store, query, and analyze DNS logs at scale using ClickHouse for threat detection and network visibility.

---

DNS logs are among the most voluminous and valuable data sources in any network environment. Every request, response, and failure carries information about user behavior, potential threats, and infrastructure health. ClickHouse excels at ingesting and querying billions of DNS records efficiently.

## Schema Design for DNS Logs

A well-structured DNS log table should capture the key fields from DNS query and response cycles:

```sql
CREATE TABLE dns_logs (
    timestamp       DateTime,
    client_ip       IPv4,
    server_ip       IPv4,
    query_name      String,
    query_type      LowCardinality(String),
    response_code   LowCardinality(String),
    response_time_ms UInt16,
    answer_count    UInt8,
    ttl             UInt32,
    is_recursive    UInt8,
    date            Date DEFAULT toDate(timestamp)
) ENGINE = MergeTree()
PARTITION BY date
ORDER BY (client_ip, timestamp);
```

Using `LowCardinality(String)` for fields like `query_type` (A, AAAA, MX, TXT) and `response_code` (NOERROR, NXDOMAIN, SERVFAIL) reduces storage and speeds filtering.

## Ingesting DNS Logs

You can ingest DNS logs from tools like `dnstap`, `bind`, or `unbound` using ClickHouse's Kafka integration or HTTP interface:

```sql
INSERT INTO dns_logs
SELECT
    parseDateTimeBestEffort(timestamp_str),
    toIPv4(client_ip_str),
    toIPv4(server_ip_str),
    query_name,
    query_type,
    response_code,
    response_time_ms,
    answer_count,
    ttl,
    is_recursive
FROM input_staging;
```

## Identifying Top Queried Domains

```sql
SELECT
    query_name,
    count() AS total_queries,
    uniq(client_ip) AS unique_clients
FROM dns_logs
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY query_name
ORDER BY total_queries DESC
LIMIT 20;
```

## Detecting NXDOMAIN Storms

A spike in NXDOMAIN responses can indicate DNS tunneling, malware C2 activity, or misconfigured applications:

```sql
SELECT
    client_ip,
    count() AS nxdomain_count
FROM dns_logs
WHERE response_code = 'NXDOMAIN'
  AND timestamp >= now() - INTERVAL 10 MINUTE
GROUP BY client_ip
HAVING nxdomain_count > 100
ORDER BY nxdomain_count DESC;
```

## Response Time Percentiles

Monitor DNS server health by tracking query latency distributions:

```sql
SELECT
    server_ip,
    quantile(0.50)(response_time_ms) AS p50,
    quantile(0.95)(response_time_ms) AS p95,
    quantile(0.99)(response_time_ms) AS p99,
    count() AS total
FROM dns_logs
WHERE date = today()
GROUP BY server_ip
ORDER BY p95 DESC;
```

## Materialized View for Hourly Summaries

To avoid scanning raw logs repeatedly, create a materialized view that pre-aggregates query volume by domain and hour:

```sql
CREATE MATERIALIZED VIEW dns_hourly_summary
ENGINE = SummingMergeTree()
ORDER BY (hour, query_name, query_type)
AS
SELECT
    toStartOfHour(timestamp) AS hour,
    query_name,
    query_type,
    count() AS queries,
    countIf(response_code = 'NXDOMAIN') AS nxdomain_count
FROM dns_logs
GROUP BY hour, query_name, query_type;
```

## Summary

ClickHouse provides the throughput and query speed needed to analyze DNS logs at network scale. By designing a compact schema with `LowCardinality` fields, building materialized views for common aggregations, and writing targeted queries for anomaly detection, you can turn raw DNS data into actionable security and performance insights with sub-second response times.
