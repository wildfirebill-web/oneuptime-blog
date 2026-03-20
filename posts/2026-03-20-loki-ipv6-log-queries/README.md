# How to Configure Loki for IPv6 Log Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Loki, LogQL, Grafana, Log Management

Description: Configure Grafana Loki to accept logs from IPv6 sources, bind its HTTP endpoint to IPv6 interfaces, and query log streams by IPv6 address fields using LogQL.

## Introduction

Grafana Loki indexes log metadata as labels and stores log content as compressed chunks. IPv6 addresses can appear as labels (for routing) or as text within log lines (for content search). Configuring Loki for IPv6 involves binding Loki's server to `::`, sending logs with IPv6 labels, and querying with LogQL.

## Step 1: Configure Loki to Listen on IPv6

```yaml
# /etc/loki/loki-config.yaml

server:
  http_listen_address: "::"     # Bind to all IPv6 interfaces
  http_listen_port: 3100
  grpc_listen_address: "::"
  grpc_listen_port: 9096

# Storage configuration (local for development)
common:
  path_prefix: /var/loki
  storage:
    filesystem:
      chunks_directory: /var/loki/chunks
      rules_directory: /var/loki/rules
  replication_factor: 1

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://[::1]:9093
```

## Step 2: Run Loki with Docker on IPv6

```bash
# Run Loki with IPv6 binding
docker run -d \
  --name loki \
  --network host \
  -v /etc/loki:/etc/loki \
  -v /var/loki:/var/loki \
  grafana/loki:3.0.0 \
  -config.file=/etc/loki/loki-config.yaml

# Verify Loki listens on IPv6
ss -tlnp | grep 3100
# Should show [::]:3100
```

## Step 3: Send Logs with IPv6 Labels

```yaml
# Promtail configuration sending to Loki over IPv6
# /etc/promtail/promtail-config.yaml

server:
  http_listen_address: "::"
  http_listen_port: 9080

clients:
  - url: http://[2001:db8::20]:3100/loki/api/v1/push

scrape_configs:
  - job_name: nginx
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx
          env: production
          __path__: /var/log/nginx/access.log
    pipeline_stages:
      - regex:
          expression: '^(?P<remote_addr>\S+) - (?P<user>\S+) \[(?P<time>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+)'
      - labels:
          remote_addr:
          method:
      - template:
          source: ip_version
          template: '{{ if contains .remote_addr ":" }}ipv6{{ else }}ipv4{{ end }}'
      - labels:
          ip_version:
```

## Step 4: LogQL Queries for IPv6

```logql
# Find all log lines from IPv6 sources
{job="nginx", ip_version="ipv6"}

# Search for a specific IPv6 address in log content
{job="nginx"} |= "2001:db8::1"

# Regex filter for any IPv6 address pattern
{job="nginx"} |~ `[0-9a-fA-F:]{3,39}:[0-9a-fA-F]{0,4}`

# Count requests by IPv6 source (with label value extraction)
sum by (remote_addr) (
  count_over_time({job="nginx", ip_version="ipv6"}[5m])
)

# Rate of IPv6 requests per minute
rate({job="nginx", ip_version="ipv6"}[1m])

# Error responses from IPv6 clients
{job="nginx", ip_version="ipv6"} |= "\" 5"

# Parse JSON logs and filter by IPv6 client
{job="app"} | json | client_ip =~ "2001:db8:.*"
```

## Step 5: Grafana Dashboard for IPv6 Loki Queries

In Grafana with a Loki data source, add these panels:

**Panel 1: IPv6 Request Rate (Time Series)**
```logql
rate({job="nginx", ip_version="ipv6"}[5m])
```

**Panel 2: Top IPv6 Sources (Table)**
```logql
topk(20, sum by (remote_addr) (
  count_over_time({job="nginx", ip_version="ipv6"}[1h])
))
```

**Panel 3: IPv6 Error Rate**
```logql
sum(rate({job="nginx", ip_version="ipv6"} |~ "\" 5[0-9][0-9] " [5m]))
/
sum(rate({job="nginx", ip_version="ipv6"}[5m]))
```

## Conclusion

Loki supports IPv6 by binding its HTTP and gRPC listeners to `::`. Use Promtail pipeline stages to extract IPv6-related labels (such as `ip_version` or `remote_addr`) for efficient LogQL stream selectors. For ad-hoc IPv6 address searches in log content, LogQL line filters (`|=`, `|~`) search the full log text. Combine stream selectors and content filters for efficient queries on high-volume log data.
