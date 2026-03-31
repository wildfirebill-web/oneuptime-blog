# How to Monitor Redis Latency Percentiles (p50, p95, p99)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Latency, Monitoring, Performance, Percentile

Description: Monitor Redis latency percentiles p50, p95, and p99 using latency stats, redis-cli tools, and Prometheus to catch tail latency issues in production.

---

Average latency hides the worst user experiences. Tracking p50, p95, and p99 latency percentiles gives you a complete view of Redis response time distribution and helps you catch tail latency spikes that averages mask.

## Enabling Latency Monitoring

Redis has a built-in latency framework. Enable it by setting a threshold (in milliseconds):

```bash
redis-cli CONFIG SET latency-monitor-threshold 1
redis-cli CONFIG SET slowlog-log-slower-than 1000
```

Then check recent latency events:

```bash
redis-cli LATENCY LATEST
redis-cli LATENCY HISTORY event
```

## Using INFO latencystats

Redis 7.0+ exposes command-level latency percentiles via `INFO latencystats`:

```bash
redis-cli INFO latencystats
```

Sample output:

```text
latency_percentiles_usec_get:p50=45,p99=120,p99.9=980
latency_percentiles_usec_set:p50=52,p99=145,p99.9=1100
```

Values are in microseconds. Divide by 1000 for milliseconds.

## Querying with Python

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

stats = r.info("latencystats")
for key, value in stats.items():
    if key.startswith("latency_percentiles_usec_"):
        cmd = key.replace("latency_percentiles_usec_", "")
        print(f"Command: {cmd}")
        # value is a dict-like string; parse it
        for percentile, usec in value.items():
            print(f"  {percentile}: {usec / 1000:.3f} ms")
```

## Prometheus and redis_exporter

`redis_exporter` (v1.44+) exposes command latency histograms when you set `--redis.latency-sampling-interval`. Use PromQL to compute percentiles:

```text
histogram_quantile(0.99, rate(redis_commands_duration_seconds_bucket[5m]))
histogram_quantile(0.95, rate(redis_commands_duration_seconds_bucket[5m]))
histogram_quantile(0.50, rate(redis_commands_duration_seconds_bucket[5m]))
```

Plot all three on a single Grafana panel to visualize the spread between median and tail latency.

## Alerting on p99 Latency

```text
alert: RedisHighP99Latency
expr: >
  histogram_quantile(0.99,
    rate(redis_commands_duration_seconds_bucket[5m])
  ) > 0.010
for: 2m
labels:
  severity: critical
annotations:
  summary: "Redis p99 latency exceeds 10ms"
```

## Common Causes of High Tail Latency

- Large values being serialized or deserialized
- Blocking commands (`BLPOP`, `BRPOP`) holding connections
- Fork latency during RDB snapshots on large datasets
- AOF fsync competing with command processing
- Network jitter between application servers and Redis

## Using redis-cli --latency

For quick interactive checks without Prometheus:

```bash
redis-cli --latency -h localhost -p 6379
redis-cli --latency-history -h localhost -p 6379
redis-cli --latency-dist -h localhost -p 6379
```

`--latency-dist` prints an ASCII heatmap of latency distribution over time.

## Summary

Tracking Redis latency percentiles (p50, p95, p99) reveals tail latency issues that averages hide. Use `INFO latencystats` for command-level percentiles in Redis 7+, `redis-cli --latency` for interactive spot checks, and Prometheus histogram quantiles for continuous monitoring and alerting in production environments.
