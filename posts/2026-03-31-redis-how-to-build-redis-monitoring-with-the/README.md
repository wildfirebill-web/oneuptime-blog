# How to Build Redis Monitoring with the TIG Stack

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Telegraf, InfluxDB, Grafana, Monitoring, TIG Stack

Description: Build a Redis monitoring solution using the TIG stack - Telegraf collects Redis metrics, InfluxDB stores them, and Grafana visualizes dashboards and alerts.

---

## What Is the TIG Stack

The TIG stack is a monitoring pipeline composed of:

- **Telegraf** - agent that collects and ships metrics from Redis (and other sources)
- **InfluxDB** - time-series database optimized for high write throughput
- **Grafana** - visualization layer with dashboard and alerting support

The TIG stack is an alternative to the Prometheus stack. InfluxDB uses a push model where Telegraf writes directly, rather than Prometheus's pull model with exporters.

## Architecture

```text
Redis Instance
     |
     v
Telegraf (redis input plugin)
     |
     v
InfluxDB (:8086)
     |
     v
Grafana (:3000)
```

## Docker Compose Setup

Create `docker-compose.yml`:

```yaml
version: "3.8"
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  influxdb:
    image: influxdb:2.7-alpine
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: adminpassword
      DOCKER_INFLUXDB_INIT_ORG: myorg
      DOCKER_INFLUXDB_INIT_BUCKET: redis
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: my-super-secret-token
    ports:
      - "8086:8086"
    volumes:
      - influxdb-data:/var/lib/influxdb2

  telegraf:
    image: telegraf:latest
    volumes:
      - ./telegraf/telegraf.conf:/etc/telegraf/telegraf.conf:ro
    depends_on:
      - redis
      - influxdb

  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - influxdb

volumes:
  influxdb-data:
  grafana-data:
```

## Telegraf Configuration

Create `telegraf/telegraf.conf`:

```toml
[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = ""
  hostname = ""
  omit_hostname = false

# Redis input plugin
[[inputs.redis]]
  servers = ["tcp://redis:6379"]
  # If Redis requires authentication:
  # password = "yourpassword"

# InfluxDB v2 output
[[outputs.influxdb_v2]]
  urls = ["http://influxdb:8086"]
  token = "my-super-secret-token"
  organization = "myorg"
  bucket = "redis"
```

## Telegraf Redis Metrics

The Telegraf redis input plugin collects metrics from `INFO all` and `INFO keyspace`. Key metrics it captures:

```text
redis_uptime_in_seconds       - How long Redis has been running
redis_connected_clients       - Number of active connections
redis_blocked_clients         - Clients waiting on blocking calls
redis_used_memory             - Memory allocated by Redis
redis_used_memory_rss         - Memory as seen by OS
redis_mem_fragmentation_ratio - Memory fragmentation ratio
redis_total_commands_processed - Commands processed since start
redis_instantaneous_ops_per_sec - Current command throughput
redis_keyspace_hits           - Cache hits
redis_keyspace_misses         - Cache misses
redis_evicted_keys            - Keys evicted due to maxmemory
redis_expired_keys            - Keys expired by TTL
redis_rdb_last_bgsave_status  - Last RDB save status
redis_aof_enabled             - Whether AOF persistence is on
```

## Setting Up InfluxDB

After starting the stack, the InfluxDB bucket is automatically initialized via the environment variables. Verify the setup:

```bash
docker-compose exec influxdb influx bucket list --org myorg --token my-super-secret-token
```

Create a retention policy if needed (InfluxDB 2.x uses retention on bucket creation):

```bash
docker-compose exec influxdb influx bucket update \
  --name redis \
  --retention 30d \
  --org myorg \
  --token my-super-secret-token
```

## Querying Redis Metrics in InfluxDB

Use Flux queries in InfluxDB or Grafana to query your Redis data:

```text
from(bucket: "redis")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "redis")
  |> filter(fn: (r) => r._field == "used_memory")
  |> aggregateWindow(every: 1m, fn: mean, createEmpty: false)
```

Hit rate calculation:

```text
hits = from(bucket: "redis")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "redis" and r._field == "keyspace_hits")
  |> last()

misses = from(bucket: "redis")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "redis" and r._field == "keyspace_misses")
  |> last()

join(tables: {hits: hits, misses: misses}, on: ["_time", "server"])
  |> map(fn: (r) => ({r with hit_rate: r._value_hits / (r._value_hits + r._value_misses)}))
```

## Grafana Dashboard Configuration

### Provision InfluxDB Data Source

Create `grafana/provisioning/datasources/influxdb.yml`:

```yaml
apiVersion: 1
datasources:
  - name: InfluxDB
    type: influxdb
    url: http://influxdb:8086
    jsonData:
      version: Flux
      organization: myorg
      defaultBucket: redis
    secureJsonData:
      token: my-super-secret-token
```

### Key Grafana Panels

**Memory Usage Panel (Flux query):**

```text
from(bucket: "redis")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "redis" and r._field == "used_memory")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
```

**Commands Per Second:**

```text
from(bucket: "redis")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "redis" and r._field == "instantaneous_ops_per_sec")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
```

**Connected Clients:**

```text
from(bucket: "redis")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "redis" and r._field == "connected_clients")
  |> aggregateWindow(every: v.windowPeriod, fn: last, createEmpty: false)
```

## Grafana Alerting

In Grafana 9+, create alerts directly from dashboard panels. Example alert for high memory:

```text
Alert name: Redis High Memory
Condition: WHEN last() OF query (A, 5m, now) IS ABOVE 0.85

Query A:
from(bucket: "redis")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "redis")
  |> filter(fn: (r) => r._field == "used_memory" or r._field == "maxmemory")
  |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> map(fn: (r) => ({r with ratio: r.used_memory / r.maxmemory}))
  |> last()
```

## Starting the Stack

```bash
docker-compose up -d
# Verify Telegraf is collecting data
docker-compose logs telegraf | grep -E "(error|redis|Connected)"
# Check data in InfluxDB
curl -XPOST "http://localhost:8086/api/v2/query?org=myorg" \
  -H "Authorization: Token my-super-secret-token" \
  -H "Content-Type: application/vnd.flux" \
  -d 'from(bucket:"redis") |> range(start: -5m) |> limit(n:5)'
```

## Summary

The TIG stack for Redis monitoring uses Telegraf's built-in Redis input plugin to push metrics directly into InfluxDB without needing a separate exporter. Configure Telegraf with the redis input and influxdb_v2 output, query data with Flux in Grafana, and set alerts on memory, eviction, and client metrics. This push-based approach simplifies the pipeline compared to Prometheus's pull model in environments where Telegraf is already deployed.
