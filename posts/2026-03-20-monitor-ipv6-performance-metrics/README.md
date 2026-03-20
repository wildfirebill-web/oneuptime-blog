# How to Monitor IPv6 Performance Metrics Over Time

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Monitoring, Prometheus, Grafana, Metrics, Performance

Description: Collect, store, and visualize IPv6 performance metrics over time using Prometheus exporters, node_exporter, and Grafana dashboards.

## Introduction

Point-in-time benchmarks are insufficient for production networks. You need continuous monitoring to detect gradual degradation, correlate incidents with performance changes, and capacity plan. Prometheus and Grafana provide an excellent stack for IPv6 performance monitoring.

## node_exporter for IPv6 Interface Metrics

```bash
# node_exporter exposes IPv6 interface counters via /proc/net/if_inet6
# Install and start
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xzf node_exporter*.tar.gz
./node_exporter --web.listen-address="[::]:9100"

# Key metrics:
# node_network_receive_bytes_total{device="eth0"}
# node_network_transmit_bytes_total{device="eth0"}
# node_network_receive_drop_total{device="eth0"}
```

## Custom IPv6 Latency Exporter

```python
#!/usr/bin/env python3
"""Custom Prometheus exporter for IPv6 latency metrics."""
from prometheus_client import start_http_server, Gauge
import subprocess
import re
import time

IPV6_LATENCY = Gauge(
    'ipv6_ping_rtt_milliseconds',
    'IPv6 ICMP round-trip time',
    ['target']
)
IPV6_LOSS = Gauge(
    'ipv6_ping_packet_loss_percent',
    'IPv6 ICMP packet loss percentage',
    ['target']
)

TARGETS = [
    "2001:4860:4860::8888",  # Google
    "2606:4700:4700::1111",  # Cloudflare
]

def measure(target: str):
    result = subprocess.run(
        ["ping6", "-c", "10", "-W", "2", target],
        capture_output=True, text=True
    )
    # Parse avg RTT
    m = re.search(r'rtt min/avg/max/mdev = [\d.]+/([\d.]+)/', result.stdout)
    if m:
        IPV6_LATENCY.labels(target=target).set(float(m.group(1)))
    # Parse packet loss
    m2 = re.search(r'(\d+)% packet loss', result.stdout)
    if m2:
        IPV6_LOSS.labels(target=target).set(float(m2.group(1)))

if __name__ == "__main__":
    start_http_server(9200, addr="::")  # Listen on all IPv6 interfaces
    while True:
        for target in TARGETS:
            measure(target)
        time.sleep(60)
```

## Prometheus Configuration

```yaml
# prometheus.yml — scrape IPv6 exporters
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets:
          - '[2001:db8::server1]:9100'
          - '[2001:db8::server2]:9100'
    # Note: brackets required for IPv6 in Prometheus targets

  - job_name: 'ipv6_latency'
    static_configs:
      - targets:
          - '[::1]:9200'  # Local latency exporter
```

## Grafana Dashboard Queries

```promql
# Average IPv6 latency to Google DNS
avg(ipv6_ping_rtt_milliseconds{target="2001:4860:4860::8888"})

# Network receive throughput for eth0 (bits/sec)
rate(node_network_receive_bytes_total{device="eth0"}[5m]) * 8

# Packet drop rate
rate(node_network_receive_drop_total{device="eth0"}[5m])

# Latency over 95th percentile (derived from histogram if available)
histogram_quantile(0.95, rate(ipv6_rtt_bucket[5m]))
```

## Alerting Rules

```yaml
# prometheus/rules/ipv6_alerts.yml
groups:
  - name: ipv6_performance
    rules:
      - alert: IPv6HighLatency
        expr: ipv6_ping_rtt_milliseconds > 100
        for: 5m
        annotations:
          summary: "IPv6 latency to {{ $labels.target }} exceeds 100ms"

      - alert: IPv6PacketLoss
        expr: ipv6_ping_packet_loss_percent > 1
        for: 2m
        annotations:
          summary: "IPv6 packet loss to {{ $labels.target }}: {{ $value }}%"
```

## Conclusion

A Prometheus + Grafana stack with custom IPv6 latency exporters provides continuous performance visibility. Set alerting thresholds based on your SLA requirements. Integrate with OneUptime to correlate network performance data with application health checks.
