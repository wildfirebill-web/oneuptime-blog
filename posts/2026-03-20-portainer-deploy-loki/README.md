# How to Deploy Loki via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Loki, Logging, Grafana, Self-Hosted

Description: Deploy Grafana Loki via Portainer as a cost-effective log aggregation system that works natively with Grafana for log exploration and correlation with metrics.

## Introduction

Grafana Loki is a horizontally scalable log aggregation system inspired by Prometheus. Unlike Elasticsearch, Loki indexes only labels (not the full log content), making it significantly cheaper to operate. This guide deploys Loki with Promtail for log collection.

## Deploy as a Stack

In Portainer, create a stack named `loki`:

```yaml
version: "3.8"

services:
  loki:
    image: grafana/loki:latest
    container_name: loki
    command: -config.file=/etc/loki/loki-config.yaml
    volumes:
      - ./loki-config.yaml:/etc/loki/loki-config.yaml:ro
      - loki_data:/loki
    ports:
      - "3100:3100"
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "wget --no-verbose --tries=1 --spider http://localhost:3100/ready || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Promtail - log shipper agent
  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    command: -config.file=/etc/promtail/config.yaml
    volumes:
      - ./promtail-config.yaml:/etc/promtail/config.yaml:ro
      # Host log access
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped

volumes:
  loki_data:
```

## Loki Configuration

Create `loki-config.yaml`:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  # Reject logs with timestamps far in the past
  reject_old_samples: true
  reject_old_samples_max_age: 168h  # 7 days

  # Retention
  retention_period: 744h   # 31 days

compactor:
  working_directory: /loki/compactor
  retention_enabled: true
  retention_delete_delay: 2h
```

## Promtail Configuration

Create `promtail-config.yaml`:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Collect Docker container logs
  - job_name: containers
    static_configs:
      - targets:
          - localhost
        labels:
          job: containerlogs
          __path__: /var/lib/docker/containers/*/*log

    pipeline_stages:
      - json:
          expressions:
            output: log
            stream: stream
            attrs:
      - json:
          expressions:
            tag: attrs.tag
          source: attrs
      - regex:
          expression: (?P<container_name>(?:[^|]*[^|]))
          source: tag
      - timestamp:
          format: RFC3339Nano
          source: time
      - labels:
          stream:
          container_name:
      - output:
          source: output

  # Collect host system logs
  - job_name: syslog
    static_configs:
      - targets:
          - localhost
        labels:
          job: syslog
          host: docker-host
          __path__: /var/log/syslog
```

## Query Logs in Grafana

After adding Loki as a data source in Grafana, use LogQL to query:

```logql
# All error logs
{job="containerlogs"} |= "ERROR"

# Nginx access logs
{container_name="nginx"} | pattern `<ip> - - [<_>] "<method> <path> <_>" <status> <_>`

# Rate of error logs per minute
rate({job="containerlogs"} |= "error" [1m])

# Log count by container
count_over_time({job="containerlogs"}[1h]) by (container_name)
```

## Integrate with Existing Grafana

Add Loki as a data source in your Grafana provisioning:

```yaml
# In grafana provisioning/datasources/datasources.yaml
datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    jsonData:
      maxLines: 1000
```

## Conclusion

Loki deployed via Portainer provides a lightweight, cost-effective log aggregation solution that integrates seamlessly with Grafana. Unlike full-text search engines, Loki's label-based indexing makes it economical to store large volumes of logs. Promtail automatically discovers and ships Docker container logs, requiring minimal configuration for immediate value.
