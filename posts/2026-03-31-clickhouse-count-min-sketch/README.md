# How to Use Count-Min Sketch in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Count-Min Sketch, Approximate Query, Streaming Analytics, Heavy Hitter

Description: Learn how to use the Count-Min Sketch in ClickHouse to estimate item frequencies with bounded error using constant memory.

---

## What Is Count-Min Sketch?

Count-Min Sketch (CMS) is a probabilistic data structure for estimating frequencies of elements in a stream. It uses a compact matrix of counters to provide frequency estimates with a bounded overestimation error. ClickHouse implements this via the `countMinSketch` aggregate function.

## Why Use Count-Min Sketch?

Exact frequency counting requires memory proportional to the number of distinct items - impractical for high-cardinality streams. Count-Min Sketch trades a small, controlled overcount error for a fixed memory footprint regardless of dataset size.

## Basic Frequency Estimation

```sql
SELECT countMinSketch(url)
FROM access_logs
WHERE toDate(timestamp) = today();
```

This returns a serialized sketch state. More usefully, combine it with `topK` for heavy-hitter detection:

```sql
SELECT
    topK(10)(url) AS top_urls,
    countMinSketch(url) AS url_sketch
FROM access_logs
GROUP BY toStartOfHour(timestamp);
```

## Sketching with Parameters

ClickHouse's CMS implementation accepts width and depth parameters that control accuracy and memory:

```sql
-- Higher width reduces error probability; higher depth reduces false positives
SELECT countMinSketch(5, 2000)(user_id)
FROM events
WHERE event_type = 'purchase';
```

- `depth`: number of hash functions (rows in the matrix)
- `width`: number of buckets per row

Error bound: with probability at least `1 - (1/2)^depth`, the estimate overshoots true frequency by at most `total_count / width`.

## Persisting Sketch State

Store sketch states in AggregatingMergeTree for incremental updates:

```sql
CREATE TABLE url_sketch_agg (
    hour    DateTime,
    sketch  AggregateFunction(countMinSketch, String)
) ENGINE = AggregatingMergeTree()
ORDER BY hour;

INSERT INTO url_sketch_agg
SELECT
    toStartOfHour(timestamp) AS hour,
    countMinSketchState(url)  AS sketch
FROM access_logs
GROUP BY hour;
```

Merge sketches across time windows:

```sql
SELECT countMinSketchMerge(sketch)
FROM url_sketch_agg
WHERE hour >= now() - INTERVAL 24 HOUR;
```

## Combining Count-Min Sketch with Bloom Filters

Use CMS alongside Bloom filter skip indexes for a layered frequency-filtering approach:

```sql
ALTER TABLE access_logs
    ADD INDEX url_bloom_idx url TYPE bloom_filter GRANULARITY 4;
```

The Bloom filter prunes granules at scan time; CMS provides frequency estimation at query time.

## Practical Use Cases

- Detecting DDoS patterns: identify IP addresses with anomalously high request counts
- Ad fraud detection: flag click sources with unexpectedly high frequency
- Log analysis: approximate frequency of error codes without full aggregation scans

```sql
-- Approximate count of each HTTP status code
SELECT
    status_code,
    countMinSketchMerge(sketch) AS approx_freq
FROM (
    SELECT status_code, countMinSketchState(request_id) AS sketch
    FROM http_logs
    GROUP BY status_code
)
GROUP BY status_code;
```

## Summary

Count-Min Sketch in ClickHouse enables memory-efficient frequency estimation over high-cardinality streams. By storing sketch states in AggregatingMergeTree tables, you can maintain running frequency estimates that are mergeable, lightweight, and queryable in real time - making them an excellent fit for anomaly detection and heavy-hitter analysis pipelines.
