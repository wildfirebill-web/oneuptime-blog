# How to Store Kubernetes Events and Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Kubernetes, Event, Metric, Observability

Description: Learn how to ingest Kubernetes events and resource metrics into ClickHouse and query pod restarts, OOM kills, and node utilization.

---

Kubernetes generates a constant stream of events - pod restarts, OOM kills, scheduling failures, and autoscaling decisions. Combined with resource metrics from kube-state-metrics and cAdvisor, these events tell the story of your cluster's health. ClickHouse is an efficient long-term store for both, with fast aggregation queries that help you spot patterns.

## Schema: Kubernetes Events

```sql
CREATE TABLE k8s_events
(
    ts           DateTime,
    namespace    LowCardinality(String),
    kind         LowCardinality(String),   -- 'Pod','Node','Deployment'
    name         String,
    reason       LowCardinality(String),   -- 'OOMKilling','BackOff','Evicted'
    message      String,
    event_type   LowCardinality(String),   -- 'Normal','Warning'
    count        UInt32
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (namespace, kind, ts);
```

## Schema: Container Metrics

```sql
CREATE TABLE k8s_container_metrics
(
    ts            DateTime,
    namespace     LowCardinality(String),
    pod           String,
    container     LowCardinality(String),
    node          LowCardinality(String),
    cpu_cores     Float32,
    mem_bytes     UInt64,
    restarts      UInt32
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (namespace, pod, ts);
```

## Pod Restart Analysis

```sql
SELECT
    namespace,
    name AS pod,
    sum(count) AS total_restarts
FROM k8s_events
WHERE reason = 'BackOff'
  AND ts >= now() - INTERVAL 24 HOUR
GROUP BY namespace, pod
ORDER BY total_restarts DESC
LIMIT 20;
```

## OOM Kill Events

```sql
SELECT
    ts,
    namespace,
    name,
    message
FROM k8s_events
WHERE reason = 'OOMKilling'
  AND ts >= now() - INTERVAL 7 DAY
ORDER BY ts DESC;
```

## CPU Utilization by Namespace

```sql
SELECT
    namespace,
    toStartOfHour(ts)  AS hour,
    avg(cpu_cores)     AS avg_cpu,
    max(cpu_cores)     AS max_cpu
FROM k8s_container_metrics
WHERE ts >= now() - INTERVAL 24 HOUR
GROUP BY namespace, hour
ORDER BY namespace, hour;
```

## Memory Usage Trend per Pod

```sql
SELECT
    pod,
    toStartOfMinute(ts) AS minute,
    avg(mem_bytes)      AS avg_mem_bytes,
    formatReadableSize(avg(mem_bytes)) AS avg_mem
FROM k8s_container_metrics
WHERE namespace = 'production'
  AND pod LIKE 'api-%'
  AND ts >= now() - INTERVAL 2 HOUR
GROUP BY pod, minute
ORDER BY pod, minute;
```

## Warning Events by Reason

```sql
SELECT
    reason,
    count()           AS events,
    uniqExact(name)   AS affected_pods
FROM k8s_events
WHERE event_type = 'Warning'
  AND ts >= now() - INTERVAL 7 DAY
GROUP BY reason
ORDER BY events DESC;
```

## Summary

ClickHouse provides a cost-effective long-term store for Kubernetes events and metrics. Store raw events and container resource samples, then query pod restart trends, OOM kills, CPU and memory utilization by namespace, and warning event distributions. This gives platform teams historical visibility that the default Kubernetes event TTL does not provide.
