# How to Deploy Prometheus via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Prometheus, Monitoring, Metrics, Self-Hosted

Description: Deploy Prometheus via Portainer with persistent storage, scrape configuration for common targets, and alert rules for infrastructure monitoring.

## Introduction

Prometheus is a leading open-source monitoring and alerting system that collects metrics from configured targets, stores them in a time-series database, and evaluates alert rules. Deploying via Portainer with configuration files makes it easy to manage scrape targets and alert rules.

## Deploy as a Stack

In Portainer, create a stack named `prometheus`:

```yaml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'   # 30-day retention
      - '--storage.tsdb.retention.size=10GB'  # Max 10GB storage
      - '--web.enable-lifecycle'              # Allow config reload via API
      - '--web.enable-admin-api'              # Enable admin API
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "wget --no-verbose --tries=1 --spider http://localhost:9090/-/healthy || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 3

  # Node Exporter for host metrics
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    network_mode: host
    restart: unless-stopped

  # cAdvisor for container metrics
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"
    restart: unless-stopped

volumes:
  prometheus_data:
```

## Prometheus Configuration

Create `prometheus.yml`:

```yaml
# prometheus.yml - global configuration
global:
  scrape_interval: 15s      # Default scrape interval
  evaluation_interval: 15s   # Rule evaluation interval
  
  # External labels added to metrics
  external_labels:
    datacenter: home-lab
    environment: production

# Alert manager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

# Load alert rules
rule_files:
  - "/etc/prometheus/rules/*.yml"

# Scrape configurations
scrape_configs:
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter (host metrics)
  - job_name: 'node'
    static_configs:
      - targets:
          - 'localhost:9100'
          - '192.168.1.11:9100'
          - '192.168.1.12:9100'

  # cAdvisor (container metrics)
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  # Application metrics via HTTP
  - job_name: 'myapp'
    metrics_path: /metrics
    static_configs:
      - targets: ['myapp:8000']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

  # Blackbox exporter (external URL monitoring)
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
          - https://api.example.com/health
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

## Alert Rules

Create `rules/host-alerts.yml`:

```yaml
groups:
  - name: host
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Host {{ $labels.instance }} is down"

      - alert: HighCPU
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}: {{ $value }}%"

      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Low disk space on {{ $labels.instance }}: {{ $value }}%"
```

## Querying Prometheus

```bash
# Query via HTTP API
curl 'http://localhost:9090/api/v1/query?query=node_memory_MemAvailable_bytes'

# Range query (last 1 hour with 15s step)
curl 'http://localhost:9090/api/v1/query_range?query=rate(http_requests_total[5m])&start=now-1h&end=now&step=15s'

# Reload configuration without restart
curl -X POST http://localhost:9090/-/reload
```

## Conclusion

Prometheus deployed via Portainer provides a robust monitoring foundation for your infrastructure. The configuration-file approach makes it easy to add new scrape targets and alert rules without container restarts (using the live reload API). Combined with Grafana for visualization and Alertmanager for notifications, it forms a complete observability stack.
