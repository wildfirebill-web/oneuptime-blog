# How to Deploy Prometheus and Grafana with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Prometheus, Grafana, Monitoring, Docker, Observability

Description: Learn how to deploy a complete Prometheus and Grafana monitoring stack using Portainer stacks for containerized infrastructure observability.

---

Prometheus scrapes metrics from targets, while Grafana provides dashboards and alerting. Deploying both as a Portainer stack gives you a production-ready monitoring platform in minutes.

---

## Deploy via Portainer Stack

In Portainer: **Stacks** → **Add stack** → paste:

```yaml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - prometheus-data:/prometheus
      - prometheus-config:/etc/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=30d"
      - "--web.enable-lifecycle"

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    ports:
      - "9093:9093"
    volumes:
      - alertmanager-config:/etc/alertmanager

volumes:
  prometheus-data:
  prometheus-config:
  grafana-data:
  alertmanager-config:
```

---

## Configure Prometheus (prometheus.yml)

Write this to the `prometheus-config` volume:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: cadvisor
    static_configs:
      - targets: ["cadvisor:8080"]
```

---

## Configure Grafana Data Source

1. Open Grafana at `http://<host>:3000`.
2. Go to **Configuration** → **Data Sources** → **Add data source**.
3. Select **Prometheus**, URL: `http://prometheus:9090`.
4. Click **Save & Test**.

---

## Import Community Dashboards

- **1860** - Node Exporter Full
- **14282** - Docker Container and Host Metrics
- **3662** - Prometheus 2.0 Stats

---

## Summary

Deploy Prometheus, Grafana, and Alertmanager as a single Portainer stack with named volumes for persistence. Configure Prometheus via `prometheus.yml` mounted into the config volume. Add Prometheus as a Grafana data source using the Docker service name (`prometheus:9090`). Import community dashboards by ID for immediate visibility into host and container metrics.
