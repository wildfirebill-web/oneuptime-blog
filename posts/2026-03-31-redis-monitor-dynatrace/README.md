# How to Monitor Redis with Dynatrace

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Dynatrace, Monitoring, Observability, APM

Description: Learn how to monitor Redis with Dynatrace - covering OneAgent deployment, Redis extension configuration, transaction tracing, and setting up alerts for Redis health.

---

Dynatrace provides automated Redis monitoring through its OneAgent, which discovers Redis processes and collects metrics automatically. For deeper observability, the Redis Extension provides detailed metrics and the Dynatrace APM captures Redis operations as spans within PurePaths.

## Deploy Dynatrace OneAgent

Install OneAgent on the host running Redis:

```bash
# Download and install OneAgent (replace with your environment ID and token)
wget -O Dynatrace-OneAgent.sh \
  "https://your-env.live.dynatrace.com/api/v1/deployment/installer/agent/unix/default/latest?Api-Token=YOUR_TOKEN"

chmod +x Dynatrace-OneAgent.sh
sudo ./Dynatrace-OneAgent.sh
```

OneAgent automatically detects Redis and begins collecting process metrics.

## Enable the Redis Extension

For detailed Redis metrics, activate the Redis Extension in Dynatrace:

1. Go to Dynatrace Hub and search for "Redis"
2. Install the "Redis" technology extension
3. Configure connection endpoints:

```yaml
# extension configuration
endpoints:
  - host: localhost
    port: 6379
    password: "${REDIS_PASSWORD}"
    tls: false
```

This enables metrics like:
- Memory usage and fragmentation
- Connected clients and rejected connections
- Keyspace hits and misses
- Command throughput per command type
- Replication lag

## Monitor Redis in PurePaths (Distributed Traces)

Dynatrace APM automatically captures Redis calls as nodes in PurePaths when using supported frameworks. No code changes required for:
- Python with `redis-py`
- Java with Jedis or Lettuce
- Node.js with `ioredis`

View Redis spans in the Dynatrace UI:
1. Navigate to Services - your service name
2. Click on any request in the Distributed Traces view
3. Redis calls appear as separate nodes showing command, host, and duration

## Set Up Anomaly Detection for Redis

Configure Dynatrace to alert on Redis anomalies:

```text
Dynatrace > Settings > Anomaly Detection > Services

Key metrics to configure:
- Response time degradation: Alert if p99 > 50ms
- Error rate increase: Alert if error rate > 5%
- Throughput drop: Alert if requests drop > 30%
```

For infrastructure-level Redis alerts:
```text
Settings > Anomaly Detection > Infrastructure

- Memory usage > 85% of maxmemory
- Connected clients > 80% of maxclients
```

## Create a Redis Dashboard

Build a custom dashboard in Dynatrace:

```text
Dashboards > Create Dashboard

Add tiles for:
1. Metric: redis.memory.used_bytes (line chart)
2. Metric: redis.clients.connected (single value)
3. Metric: redis.keyspace.hits / (hits + misses) (gauge)
4. Metric: redis.commands.calls.persecond (bar chart)
5. Problem tile: filtered to Redis processes
```

## Use the Dynatrace API for Custom Metrics

Push custom Redis metrics via the Dynatrace Metrics API:

```python
import requests

def push_cache_hit_rate(hit_rate: float, environment: str):
    url = f"https://your-env.live.dynatrace.com/api/v2/metrics/ingest"
    headers = {
        "Authorization": f"Api-Token {DYNATRACE_TOKEN}",
        "Content-Type": "text/plain"
    }
    payload = f"app.redis.cache_hit_rate,environment={environment} {hit_rate}"
    requests.post(url, headers=headers, data=payload)
```

## Summary

Dynatrace monitors Redis at multiple levels: process discovery via OneAgent, detailed metrics via the Redis Extension, and application-level tracing via PurePaths. Anomaly detection automatically baselines normal behavior and alerts on deviations without manual threshold configuration. This AI-assisted approach reduces alert noise and helps identify real performance degradation in Redis-backed applications.
