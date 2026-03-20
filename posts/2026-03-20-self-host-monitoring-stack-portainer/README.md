# How to Self-Host a Monitoring Stack with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Prometheus, Grafana, Monitoring, Docker Compose, Metrics, Self-Hosting

Description: Learn how to deploy a complete self-hosted monitoring stack with Prometheus, Grafana, and cAdvisor via Portainer for infrastructure and container metrics.

---

A self-hosted monitoring stack gives you full visibility into your infrastructure without sending data to third-party services. This guide deploys Prometheus (metrics collection), cAdvisor (container metrics), and Grafana (visualization) via Portainer.

## Compose Stack

```yaml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:latest
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=30d    # Keep 30 days of metrics

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    restart: unless-stopped
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    ports:
      - "8098:8080"

  node-exporter:
    image: prom/node-exporter:latest
    restart: unless-stopped
    # Expose host system metrics (CPU, memory, disk)
    network_mode: host
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys

  grafana:
    image: grafana/grafana:latest
    restart: unless-stopped
    ports:
      - "3200:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: adminpass  # Change this
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  prometheus_data:
  grafana_data:
```

## Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
```

## Grafana Dashboard Setup

1. Open `http://<host>:3200` and log in.
2. Add Prometheus as a data source: `http://prometheus:9090`.
3. Import community dashboards:
   - **1860** — Node Exporter Full (host metrics)
   - **14282** — cAdvisor Exporter (container metrics)

## Monitoring Your Monitoring Stack

Use OneUptime to monitor `http://<host>:9090/-/healthy` and `http://<host>:3200/api/health`. If your monitoring stack goes down, you need an external alert — this is exactly where OneUptime adds value.
