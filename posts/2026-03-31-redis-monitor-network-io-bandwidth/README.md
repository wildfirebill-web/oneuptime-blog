# How to Monitor Redis Network I/O and Bandwidth

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Monitoring, Network, Bandwidth, Observability

Description: Learn how to track Redis network I/O and bandwidth using INFO stats metrics, Prometheus exporters, and alerting on unusual traffic patterns.

---

Redis serializes all command input and output over TCP. Monitoring network bandwidth helps you detect runaway clients, oversized payloads, and capacity limits before your server becomes a bottleneck.

## Key Metrics in INFO stats

The `INFO stats` section exposes bytes sent and received since startup:

```bash
redis-cli INFO stats | grep -E "total_net_input_bytes|total_net_output_bytes|instantaneous_input_kbps|instantaneous_output_kbps"
```

Sample output:

```text
total_net_input_bytes:5823491072
total_net_output_bytes:12048293847
instantaneous_input_kbps:312.45
instantaneous_output_kbps:748.20
```

`instantaneous_input_kbps` and `instantaneous_output_kbps` give you a live rolling average without any calculation.

## Computing Bandwidth Over an Interval

To measure average bandwidth over a custom window, take two snapshots:

```bash
#!/bin/bash
INTERVAL=10
IN1=$(redis-cli INFO stats | grep total_net_input_bytes | awk -F: '{print $2}' | tr -d '\r')
OUT1=$(redis-cli INFO stats | grep total_net_output_bytes | awk -F: '{print $2}' | tr -d '\r')
sleep $INTERVAL
IN2=$(redis-cli INFO stats | grep total_net_input_bytes | awk -F: '{print $2}' | tr -d '\r')
OUT2=$(redis-cli INFO stats | grep total_net_output_bytes | awk -F: '{print $2}' | tr -d '\r')
echo "Avg input kbps:  $(( (IN2 - IN1) * 8 / INTERVAL / 1024 ))"
echo "Avg output kbps: $(( (OUT2 - OUT1) * 8 / INTERVAL / 1024 ))"
```

## Python Bandwidth Collector

```python
import redis
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def bandwidth_kbps(interval: int = 5):
    s1 = r.info("stats")
    time.sleep(interval)
    s2 = r.info("stats")
    in_kbps = (s2["total_net_input_bytes"] - s1["total_net_input_bytes"]) * 8 / interval / 1024
    out_kbps = (s2["total_net_output_bytes"] - s1["total_net_output_bytes"]) * 8 / interval / 1024
    return round(in_kbps, 2), round(out_kbps, 2)

while True:
    rx, tx = bandwidth_kbps()
    print(f"RX: {rx} kbps  TX: {tx} kbps")
```

## Prometheus and Grafana

`redis_exporter` exposes counters that map directly to these fields. Use PromQL to compute rates:

```text
rate(redis_net_input_bytes_total[1m]) * 8 / 1024
rate(redis_net_output_bytes_total[1m]) * 8 / 1024
```

Create a Grafana panel with both series on the same graph to visualize asymmetric traffic - large output relative to input usually means clients are fetching large values or running expensive SCAN operations.

## Alerting on Bandwidth Spikes

```text
alert: RedisHighNetworkOutput
expr: rate(redis_net_output_bytes_total[1m]) > 104857600
for: 3m
labels:
  severity: warning
annotations:
  summary: "Redis outbound bandwidth exceeds 100 MB/s"
```

## Common Causes of High Bandwidth

- Fetching large strings or uncompressed binary values
- Running `KEYS *` or unbounded `SCAN` in production
- Pub/Sub channels with many subscribers receiving large messages
- Replication sending full `RDB` snapshots to replicas after disconnections

## Summary

Redis network I/O metrics (`instantaneous_input_kbps`, `instantaneous_output_kbps`, and the total byte counters) give you a clear view of bandwidth consumption. Collect them via redis-cli polling, a Python script, or Prometheus/Grafana for long-term trending. Set alerts on both input and output spikes to catch runaway clients and capacity issues early.
