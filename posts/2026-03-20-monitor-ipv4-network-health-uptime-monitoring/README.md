# How to Monitor IPv4 Network Health with Uptime Monitoring Tools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Uptime Monitoring, Network Health, Prometheus, Alerting, Availability

Description: Monitor IPv4 network health and service availability using Prometheus blackbox exporter, Uptime Kuma, and custom shell scripts for ICMP, TCP, and HTTP checks.

## Introduction

IPv4 network health monitoring covers ICMP reachability, TCP port availability, and HTTP endpoint health. A layered approach — active probing at regular intervals with alerting on failure — provides early warning before users are impacted.

## Prometheus Blackbox Exporter

```yaml
# blackbox.yml
modules:
  icmp_check:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: ip4

  tcp_connect:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: ip4

  http_2xx:
    prober: http
    timeout: 10s
    http:
      method: GET
      valid_status_codes: [200, 204]
      preferred_ip_protocol: ip4
      follow_redirects: true
```

```yaml
# prometheus.yml scrape config
scrape_configs:
  - job_name: blackbox_icmp
    metrics_path: /probe
    params:
      module: [icmp_check]
    static_configs:
      - targets:
          - 10.1.1.1      # Core router
          - 10.1.10.1     # LAN gateway
          - 8.8.8.8       # Internet reachability
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115

  - job_name: blackbox_http
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - http://192.168.1.100/health
          - https://api.example.com/health
```

## Alertmanager Rules

```yaml
groups:
  - name: network_health
    rules:
      - alert: ICMPProbeFailed
        expr: probe_success{job="blackbox_icmp"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "ICMP probe failed for {{ $labels.instance }}"
          description: "Host {{ $labels.instance }} is unreachable"

      - alert: HTTPProbeFailed
        expr: probe_success{job="blackbox_http"} == 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "HTTP check failed for {{ $labels.instance }}"

      - alert: HighLatency
        expr: probe_duration_seconds{job="blackbox_icmp"} > 0.1
        for: 5m
        annotations:
          summary: "High latency to {{ $labels.instance }}: {{ $value }}s"
```

## Shell-Based ICMP Monitor

```bash
#!/bin/bash
# monitor_ipv4.sh — simple ping-based monitor

HOSTS=(
  "10.1.1.1:Core-Router"
  "10.1.10.1:LAN-GW"
  "8.8.8.8:Google-DNS"
)

for entry in "${HOSTS[@]}"; do
  ip="${entry%%:*}"
  name="${entry##*:}"
  if ping -c 2 -W 2 "$ip" > /dev/null 2>&1; then
    echo "[UP]   $name ($ip)"
  else
    echo "[DOWN] $name ($ip)" >&2
    # Send alert (webhook, email, etc.)
    curl -s -X POST "$SLACK_WEBHOOK" \
      -H 'Content-Type: application/json' \
      -d "{\"text\":\"ALERT: $name ($ip) is DOWN\"}"
  fi
done
```

## Uptime Kuma Configuration

Uptime Kuma (self-hosted uptime monitor) supports:
- **Ping monitors**: ICMP to IPv4 hosts
- **TCP port monitors**: Check port open/closed
- **HTTP monitors**: Check HTTP status codes
- **DNS monitors**: Verify A record resolution

```bash
# Run Uptime Kuma via Docker
docker run -d \
  --name uptime-kuma \
  -p 3001:3001 \
  -v uptime-kuma:/app/data \
  louislam/uptime-kuma:1

# Access at http://your-server:3001
```

## Key Metrics to Monitor

```
Network Layer:
  ICMP RTT to gateway:        < 5ms (LAN), < 50ms (WAN)
  Packet loss to gateway:     = 0%
  ICMP to 8.8.8.8:           < 50ms (internet reachability)

Application Layer:
  HTTP response time:         < 500ms
  HTTP success rate:          > 99.9%
  TCP connection time:        < 100ms

Infrastructure:
  Interface errors/drops:     = 0
  CPU on routers/firewalls:   < 70%
```

## Conclusion

IPv4 network health monitoring uses active probing (ICMP, TCP, HTTP) with the Prometheus blackbox exporter or Uptime Kuma for visualization. Set alerting thresholds for packet loss, latency, and HTTP failures with short evaluation windows (1–2 minutes) to detect outages before users escalate. Combine with passive monitoring (SNMP interface counters, flow data) for a complete picture.
