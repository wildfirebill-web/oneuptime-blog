# How to Build a Complete Redis Monitoring Stack

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Prometheus, Grafana, AlertManager, Monitoring, Observability

Description: Set up a full Redis monitoring stack using Prometheus, Grafana, and AlertManager with redis_exporter to collect metrics, visualize dashboards, and fire alerts.

---

## Overview

A complete Redis monitoring stack involves three components working together:

- **Prometheus** - scrapes and stores time-series metrics from Redis via redis_exporter
- **Grafana** - visualizes Redis metrics with pre-built dashboards
- **AlertManager** - routes and deduplicates alerts from Prometheus rules

This guide covers deploying all three with Docker Compose, configuring Redis scraping, setting up dashboards, and writing alert rules.

## Architecture

```text
Redis Instance
     |
     v
redis_exporter (:9121)
     |
     v
Prometheus (:9090) --> AlertManager (:9093)
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
    command: redis-server --save 60 1 --loglevel warning

  redis-exporter:
    image: oliver006/redis_exporter:latest
    environment:
      REDIS_ADDR: redis://redis:6379
    ports:
      - "9121:9121"
    depends_on:
      - redis

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules:/etc/prometheus/rules
    ports:
      - "9090:9090"
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=30d"

  alertmanager:
    image: prom/alertmanager:latest
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"

  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

volumes:
  grafana-data:
```

## Prometheus Configuration

Create `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - "rules/redis_alerts.yml"

scrape_configs:
  - job_name: "redis"
    static_configs:
      - targets:
          - redis-exporter:9121
    scrape_interval: 10s
```

## Alert Rules

Create `prometheus/rules/redis_alerts.yml`:

```yaml
groups:
  - name: redis
    rules:
      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis instance is down"
          description: "Redis at {{ $labels.instance }} has been down for more than 1 minute."

      - alert: RedisHighMemoryUsage
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage above 85%"
          description: "Redis memory is at {{ $value | humanizePercentage }} on {{ $labels.instance }}."

      - alert: RedisHighConnectedClients
        expr: redis_connected_clients > 500
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis has too many connected clients"
          description: "Redis has {{ $value }} connected clients on {{ $labels.instance }}."

      - alert: RedisRejectedConnections
        expr: increase(redis_rejected_connections_total[5m]) > 0
        labels:
          severity: critical
        annotations:
          summary: "Redis is rejecting connections"

      - alert: RedisHighKeyEvictions
        expr: increase(redis_evicted_keys_total[5m]) > 100
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Redis is evicting keys"
          description: "{{ $value }} keys were evicted in the last 5 minutes on {{ $labels.instance }}."

      - alert: RedisSlowlogEntries
        expr: increase(redis_slowlog_length[5m]) > 0
        labels:
          severity: info
        annotations:
          summary: "Redis slow log has new entries"
```

## AlertManager Configuration

Create `alertmanager/alertmanager.yml`:

```yaml
global:
  smtp_smarthost: "smtp.example.com:587"
  smtp_from: "alerts@example.com"
  smtp_auth_username: "alerts@example.com"
  smtp_auth_password: "yourpassword"

route:
  group_by: ["alertname", "instance"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: "email-alerts"
  routes:
    - match:
        severity: critical
      receiver: "pagerduty-critical"

receivers:
  - name: "email-alerts"
    email_configs:
      - to: "ops-team@example.com"

  - name: "pagerduty-critical"
    pagerduty_configs:
      - routing_key: "your-pagerduty-integration-key"
```

## Grafana Dashboard Setup

### Automatic Provisioning

Create `grafana/provisioning/datasources/prometheus.yml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
```

Create `grafana/provisioning/dashboards/dashboards.yml`:

```yaml
apiVersion: 1
providers:
  - name: "Redis"
    orgId: 1
    folder: "Redis"
    type: file
    options:
      path: /etc/grafana/provisioning/dashboards
```

### Import Pre-built Dashboard

The most popular Redis dashboard for Grafana is dashboard ID **11835** (Redis Exporter). Import it via the Grafana UI or via API:

```bash
curl -X POST http://admin:admin@localhost:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  -d '{"inputs":[{"name":"DS_PROMETHEUS","type":"datasource","pluginId":"prometheus","value":"Prometheus"}],"folderId":0,"overwrite":true,"path":"11835"}'
```

## Starting the Stack

```bash
docker-compose up -d
```

Verify all services are healthy:

```bash
docker-compose ps
curl -s http://localhost:9121/metrics | grep redis_up
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep health
```

Access Grafana at `http://localhost:3000` with `admin/admin`.

## Key Metrics to Monitor

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `redis_up` | Instance availability | == 0 |
| `redis_memory_used_bytes` | Memory used by Redis | > 85% of max |
| `redis_connected_clients` | Active connections | > 500 |
| `redis_commands_total` | Command throughput | baseline deviation |
| `redis_keyspace_hits_total` | Cache hits | hit rate < 90% |
| `redis_evicted_keys_total` | Evicted keys | > 0 |

## Summary

This Redis monitoring stack uses redis_exporter to expose Prometheus-compatible metrics, Prometheus to scrape and evaluate alert rules, AlertManager to route notifications via email or PagerDuty, and Grafana for visualization. Deploy with Docker Compose, import dashboard 11835 for an instant Redis overview, and tune alert thresholds based on your production baselines.
