# How to Build Real-Time Search Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Search Analytics, Real-Time Analytics, Materialized View, Analytics

Description: Track search queries, zero-result rates, and click-through rates in real time using ClickHouse materialized views and fast aggregation queries.

---

Search analytics answers questions like "What are users searching for?", "Which queries return no results?", and "What is the click-through rate per query term?". ClickHouse handles this at scale with millisecond query times.

## Event Schema

```sql
CREATE TABLE search_events
(
    session_id  UUID,
    user_id     UInt64,
    query       String,
    result_count UInt32,
    clicked     UInt8,  -- 1 if user clicked a result
    ts          DateTime DEFAULT now()
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (ts, query);
```

## Zero-Result Rate View

```sql
CREATE MATERIALIZED VIEW zero_result_mv
ENGINE = SummingMergeTree()
ORDER BY (query, day)
AS
SELECT
    query,
    toDate(ts) AS day,
    count()               AS total,
    countIf(result_count = 0) AS zero_results
FROM search_events
GROUP BY query, day;
```

## Top Queries and CTR

```sql
SELECT
    query,
    sum(total) AS searches,
    sum(zero_results) AS no_results,
    round(sum(zero_results) / sum(total) * 100, 2) AS zero_result_pct
FROM zero_result_mv
WHERE day >= today() - 7
GROUP BY query
ORDER BY searches DESC
LIMIT 20;
```

For click-through rate, add a separate view tracking clicks:

```sql
CREATE MATERIALIZED VIEW search_ctr_mv
ENGINE = SummingMergeTree()
ORDER BY (query, day)
AS
SELECT
    query,
    toDate(ts) AS day,
    count()            AS impressions,
    sum(clicked)       AS clicks
FROM search_events
GROUP BY query, day;
```

```sql
SELECT
    query,
    sum(impressions) AS impressions,
    sum(clicks) AS clicks,
    round(sum(clicks) / sum(impressions) * 100, 2) AS ctr_pct
FROM search_ctr_mv
WHERE day = today()
GROUP BY query
ORDER BY impressions DESC
LIMIT 50;
```

## Real-Time Dashboard

Use the ClickHouse HTTP interface to feed a Grafana dashboard. Set a 10-second refresh for live search trend panels:

```bash
curl -s "http://localhost:8123/?query=SELECT+query,+sum(total)+FROM+zero_result_mv+WHERE+day%3Dtoday()+GROUP+BY+query+ORDER+BY+2+DESC+LIMIT+10+FORMAT+JSON"
```

## Summary

ClickHouse materialized views maintain pre-aggregated search metrics with no additional infrastructure. Pair them with Grafana for a real-time search analytics dashboard that shows query trends, zero-result rates, and CTR at a glance.
