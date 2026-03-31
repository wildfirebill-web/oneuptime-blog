# How to Monitor Redis Cloud Instances

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redis Cloud, Monitoring, Observability, Alerting

Description: Monitor Redis Cloud instances using built-in metrics, the Redis Cloud API, and Prometheus integration to track memory, latency, and throughput.

---

Redis Cloud provides built-in monitoring through its console, but production workloads need deeper visibility - custom alerts, historical trends, and integration with your existing observability stack. This guide covers the available monitoring options.

## Built-in Console Metrics

In the Redis Cloud console, click on a database and navigate to **Metrics**. Available charts include:

- **Memory usage** - used vs allocated
- **Operations/sec** - read and write ops
- **Network throughput** - bytes in/out
- **Connections** - current connected clients
- **Keyspace** - number of keys

These are visible for the past hour, day, or week.

## Setting Up Alerts in Redis Cloud Console

Under **Database Settings - Alerts**:

```text
Alert type: Memory usage
Threshold: 80%
Action: Email notification

Alert type: Connections
Threshold: 500
Action: Email notification

Alert type: Replication lag
Threshold: 5 seconds
Action: Email notification
```

## Using the Redis Cloud API

Redis Cloud exposes a REST API for programmatic monitoring:

```bash
# Get the database's current metrics
curl -s -X GET \
  "https://api.redislabs.com/v1/subscriptions/<sub-id>/databases/<db-id>/stats" \
  -H "accept: application/json" \
  -H "x-api-key: <api-key>" \
  -H "x-secret-key: <secret-key>" | jq .
```

Key fields in the response:

```json
{
  "instantaneousOpsPerSec": 1250,
  "usedMemory": 524288000,
  "memoryUsagePercent": 51.2,
  "connectedClients": 42,
  "keyCount": 150000
}
```

## Polling Metrics with a Script

```python
import requests
import time

API_KEY = "your-api-key"
SECRET_KEY = "your-secret-key"
SUB_ID = "12345"
DB_ID = "67890"

headers = {
    "x-api-key": API_KEY,
    "x-secret-key": SECRET_KEY,
}

def get_metrics():
    url = f"https://api.redislabs.com/v1/subscriptions/{SUB_ID}/databases/{DB_ID}/stats"
    response = requests.get(url, headers=headers)
    data = response.json()
    return {
        "ops_per_sec": data.get("instantaneousOpsPerSec", 0),
        "memory_percent": data.get("memoryUsagePercent", 0),
        "connections": data.get("connectedClients", 0),
    }

while True:
    metrics = get_metrics()
    print(metrics)
    if metrics["memory_percent"] > 85:
        print("WARNING: Memory above 85%")
    time.sleep(60)
```

## Prometheus Integration

Redis Cloud supports Prometheus scraping for Flexible plan subscribers. Enable it from **Database Settings - Prometheus Integration** and add to your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: redis_cloud
    static_configs:
      - targets: ["metrics.redis-cloud.example.com:8070"]
    metrics_path: /metrics
    scheme: https
    tls_config:
      insecure_skip_verify: false
```

Key Prometheus metrics:

```text
redis_cloud_db_memory_used_bytes
redis_cloud_db_instantaneous_ops_per_sec
redis_cloud_db_connected_clients
redis_cloud_db_keyspace_hits_total
redis_cloud_db_keyspace_misses_total
```

## Grafana Dashboard

After configuring Prometheus, import a Grafana dashboard. A sample panel for cache hit rate:

```text
Panel: Cache Hit Rate
Query: rate(redis_cloud_db_keyspace_hits_total[5m]) /
       (rate(redis_cloud_db_keyspace_hits_total[5m]) +
        rate(redis_cloud_db_keyspace_misses_total[5m]))
Unit: Percent (0-100)
Alert: < 90% for 5 minutes
```

## Summary

Monitor Redis Cloud instances using the built-in console for quick visibility, the REST API for programmatic polling, and Prometheus + Grafana for production-grade dashboards. Configure memory and replication lag alerts in the console as a minimum baseline, and track cache hit rate as the primary health indicator for caching workloads.
