# How to Monitor SRv6 Paths and Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SRv6, Monitoring, Performance, traceroute, Prometheus, Networking

Description: Monitor SRv6 path health and performance using traceroute, ping, Prometheus metrics, and custom path probing to detect SID failures and latency issues.

## Introduction

SRv6 paths must be monitored at multiple levels: individual SID reachability, end-to-end latency through a segment list, and per-hop performance. This guide covers monitoring tools and metric collection strategies.

## Step 1: SID Reachability Monitoring

```bash
# Ping each SID in a path to check reachability
SIDS=(
    "5f00:1:2:0:e001::"   # Waypoint 1
    "5f00:2:3:0:e001::"   # Waypoint 2
    "5f00:3:1:0:e000::"   # Destination
)

for sid in "${SIDS[@]}"; do
    if ping6 -c 3 -W 2 "$sid" > /dev/null 2>&1; then
        echo "OK: $sid"
    else
        echo "UNREACHABLE: $sid"
    fi
done
```

## Step 2: End-to-End Path Tracing with traceroute

```bash
# Trace path through SRv6 — requires kernel support for ICMP replies at SIDs
traceroute6 5f00:3:1::1

# Use ICMP type 3 (time exceeded) to measure per-hop RTT
# Each SRv6 waypoint should appear as a hop

# For SRv6 TE policy path verification
traceroute6 -s 5f00:1:1::1 5f00:3:1::1
```

## Step 3: SRv6 Statistics Collection

```bash
# Linux: check per-route SRv6 statistics
ip -6 route show | grep seg6

# Get detailed stats for a seg6local route
# (requires eBPF or netlink statistics)
ip -6 -s route show | grep seg6local

# Cisco IOS-XR: show per-policy SRv6 statistics
# show segment-routing srv6 forwarding
# show segment-routing traffic-eng policy detail
```

## Step 4: Custom Prometheus Exporter for SRv6

```python
#!/usr/bin/env python3
# srv6_exporter.py — Prometheus metrics for SRv6 path health

import subprocess
import re
import time
from prometheus_client import start_http_server, Gauge, Counter

# Metrics
srv6_sid_reachable = Gauge(
    "srv6_sid_reachable",
    "SRv6 SID reachability (1=reachable, 0=unreachable)",
    ["sid", "function"]
)
srv6_sid_rtt_ms = Gauge(
    "srv6_sid_rtt_ms",
    "SRv6 SID ping RTT in milliseconds",
    ["sid"]
)
srv6_path_latency_ms = Gauge(
    "srv6_path_latency_ms",
    "End-to-end latency through SRv6 path",
    ["path_name", "endpoint"]
)

# SID definitions
SIDS = [
    {"addr": "5f00:1:2:0:e001::", "function": "End.X"},
    {"addr": "5f00:2:3:0:e001::", "function": "End.X"},
    {"addr": "5f00:3:1:0:e000::", "function": "End.DT6"},
]

def probe_sid(sid: dict) -> tuple:
    """Returns (reachable, avg_rtt_ms)."""
    result = subprocess.run(
        ["ping6", "-c", "5", "-W", "2", "-q", sid["addr"]],
        capture_output=True, text=True, timeout=15
    )
    output = result.stdout + result.stderr

    # Parse loss
    loss_match = re.search(r"(\d+)% packet loss", output)
    loss = int(loss_match.group(1)) if loss_match else 100
    reachable = 1 if loss < 100 else 0

    # Parse RTT
    rtt_match = re.search(r"rtt .+ = [\d.]+/([\d.]+)/", output)
    avg_rtt = float(rtt_match.group(1)) if rtt_match else 0.0

    return reachable, avg_rtt


def collect_metrics():
    for sid in SIDS:
        try:
            reachable, rtt = probe_sid(sid)
            srv6_sid_reachable.labels(
                sid=sid["addr"], function=sid["function"]
            ).set(reachable)
            srv6_sid_rtt_ms.labels(sid=sid["addr"]).set(rtt)
        except Exception as e:
            print(f"Error probing {sid['addr']}: {e}")


if __name__ == "__main__":
    start_http_server(9300)
    print("SRv6 exporter on :9300")
    while True:
        collect_metrics()
        time.sleep(30)
```

## Step 5: Grafana Dashboard Queries

```promql
# SID availability percentage
avg_over_time(srv6_sid_reachable{function="End.DT6"}[5m]) * 100

# Average RTT to each SID
srv6_sid_rtt_ms

# Alert: SID unreachable for > 2 minutes
srv6_sid_reachable == 0
```

## Step 6: Prometheus Alert Rules

```yaml
groups:
  - name: srv6-monitoring
    rules:
      - alert: SRv6SIDUnreachable
        expr: srv6_sid_reachable == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "SRv6 SID {{ $labels.sid }} ({{ $labels.function }}) is unreachable"

      - alert: SRv6HighLatency
        expr: srv6_sid_rtt_ms > 50
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency to SRv6 SID {{ $labels.sid }}: {{ $value }}ms"
```

## Conclusion

SRv6 path monitoring requires per-SID reachability checks and end-to-end latency measurement. Custom Prometheus exporters fill the gap where native SRv6 monitoring support is limited. Use OneUptime alongside these metrics to provide end-user perspective health checks through SRv6 service chains.
