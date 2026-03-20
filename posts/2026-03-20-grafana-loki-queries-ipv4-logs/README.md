# How to Query IPv4 Access Logs with Grafana Loki

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Grafana, Loki, IPv4, Log Queries, LogQL, Monitoring

Description: Use Grafana Loki to collect, query, and visualize IPv4 address patterns in web server access logs, build log panels in Grafana, and create alerts on log-based metrics.

## Introduction

Loki stores and queries logs using LogQL. For IPv4 monitoring, you can filter logs by client IP address, count requests per source IP, identify top talkers, and alert on suspicious access patterns—all without indexing log content.

## Loki Configuration for Log Collection

```yaml
# /etc/loki/loki-local-config.yaml

auth_enabled: false

server:
  http_listen_address: 10.0.0.25
  http_listen_port: 3100
  grpc_listen_address: 10.0.0.25

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v12
      index:
        prefix: index_
        period: 24h

storage_config:
  tsdb_shipper:
    active_index_directory: /var/loki/index
    cache_location: /var/loki/index_cache
  filesystem:
    directory: /var/loki/chunks
```

## Promtail Configuration (Log Shipper)

```yaml
# /etc/promtail/config.yml

server:
  http_listen_address: 10.0.0.1
  http_listen_port: 9080

clients:
  - url: http://10.0.0.25:3100/loki/api/v1/push

scrape_configs:
  - job_name: nginx_access
    static_configs:
      - targets: ['localhost']
        labels:
          job: nginx
          host: 10.0.0.1
          __path__: /var/log/nginx/access.log

  - job_name: syslog
    static_configs:
      - targets: ['localhost']
        labels:
          job: syslog
          host: 10.0.0.1
          __path__: /var/log/syslog
```

## LogQL Queries for IPv4 Analysis

```logql
# All Nginx access logs
{job="nginx"}

# Requests from specific IPv4
{job="nginx"} |= "203.0.113.20"

# Requests matching IPv4 pattern (regex)
{job="nginx"} |~ `\b10\.0\.0\.\d+\b`

# Count requests per source IP (metric query)
sum by (remote_addr) (
  count_over_time({job="nginx"} | logfmt | line_format "{{.remote_addr}}" [5m])
)

# Top 10 IPs by request count
topk(10,
  sum by (remote_addr) (
    count_over_time({job="nginx"} | regexp `(?P<remote_addr>\d+\.\d+\.\d+\.\d+)` [5m])
  )
)

# HTTP 4xx errors per IP
{job="nginx"} | regexp `(?P<remote_addr>\d+\.\d+\.\d+\.\d+) .* (?P<status>[45]\d{2})`
| status =~ "4.."

# Bandwidth per IP (if log includes bytes)
sum by (remote_addr) (
  sum_over_time(
    {job="nginx"} | logfmt | unwrap bytes_sent [5m]
  )
)
```

## Grafana Panel Configuration

```bash
# In Grafana, create a new panel with Loki data source:

# Panel 1: Log stream (Logs type)
# Query: {job="nginx"} | logfmt | remote_addr != ""

# Panel 2: Request rate by IP (Time series type)
# Query: sum by (remote_addr) (rate({job="nginx"} | logfmt [5m]))

# Panel 3: Top IPs table (Table type)
# Query: topk(10, sum by (remote_addr) (count_over_time({job="nginx"}[1h])))

# Panel 4: Error rate (Stat type)
# Query: sum(count_over_time({job="nginx"} |= "\" 5" [5m]))
```

## Loki Alert Rules

```yaml
# /etc/loki/rules/nginx_alerts.yml

groups:
  - name: nginx_logs
    rules:
      - alert: HighErrorRate
        expr: >
          sum(rate({job="nginx"} |~ " (4|5)[0-9][0-9] " [5m])) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High HTTP error rate in nginx logs"
```

## Conclusion

Loki with LogQL enables IPv4-based log analysis without indexing full log content. Use `|=` for literal IP search, `|~` for regex IP patterns, and `| regexp` with named captures for extracting IPs into labels. Combine with Prometheus metrics for mixed log+metric dashboards. Alert on log-derived metrics using Loki ruler configuration for detecting high error rates or suspicious access patterns.
