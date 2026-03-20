# How to Deploy the Grafana-Prometheus-Loki Stack via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Grafana, Prometheus, Loki, Observability, Self-Hosted

Description: Deploy the complete Grafana observability stack (Grafana, Prometheus, and Loki) via Portainer for unified metrics and log monitoring in a single stack deployment.

## Introduction

Grafana, Prometheus, and Loki form the modern open-source observability stack: Prometheus for metrics collection, Loki for log aggregation, and Grafana for unified visualization of both. This guide deploys all three plus supporting agents as a single Portainer stack.

## Deploy the Complete Stack

In Portainer, create a stack named `observability`:

```yaml
version: "3.8"

services:
  # Prometheus - metrics collection and storage
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./alert-rules.yml:/etc/prometheus/alert-rules.yml:ro
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - observability
    restart: unless-stopped

  # Loki - log aggregation
  loki:
    image: grafana/loki:latest
    container_name: loki
    command: -config.file=/etc/loki/config.yaml
    volumes:
      - ./loki-config.yaml:/etc/loki/config.yaml:ro
      - loki_data:/loki
    ports:
      - "3100:3100"
    networks:
      - observability
    restart: unless-stopped

  # Grafana - unified visualization
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin_password
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
      - loki
    networks:
      - observability
    restart: unless-stopped

  # Node Exporter - host metrics
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
    network_mode: host
    restart: unless-stopped

  # cAdvisor - container metrics
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
    networks:
      - observability
    restart: unless-stopped

  # Promtail - log shipper
  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    command: -config.file=/etc/promtail/config.yaml
    volumes:
      - ./promtail-config.yaml:/etc/promtail/config.yaml:ro
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - observability
    restart: unless-stopped

  # Alertmanager - alert routing
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    command:
      - '--config.file=/etc/alertmanager/config.yml'
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/config.yml:ro
    ports:
      - "9093:9093"
    networks:
      - observability
    restart: unless-stopped

networks:
  observability:
    driver: bridge

volumes:
  prometheus_data:
  loki_data:
  grafana_data:
```

## Grafana Provisioning

Create `grafana/provisioning/datasources/all.yaml`:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
    access: proxy

  - name: Loki
    type: loki
    url: http://loki:3100
    access: proxy
```

Create `grafana/provisioning/dashboards/dashboards.yaml`:

```yaml
apiVersion: 1

providers:
  - name: Default
    type: file
    options:
      path: /var/lib/grafana/dashboards
    updateIntervalSeconds: 30
```

## Prometheus Configuration

Create `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "/etc/prometheus/alert-rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

## Accessing the Stack

After deployment:

| Service | URL |
|---------|-----|
| Grafana | `http://<host>:3000` |
| Prometheus | `http://<host>:9090` |
| Alertmanager | `http://<host>:9093` |
| cAdvisor | `http://<host>:8080` |

Log in to Grafana with `admin/admin_password`, then import dashboard ID **1860** for Node Exporter metrics and explore your container logs from the Explore tab using Loki.

## Conclusion

The Grafana-Prometheus-Loki observability stack deployed via Portainer provides unified metrics and log monitoring in a single coherent deployment. Auto-provisioned data sources mean Grafana is connected to both Prometheus and Loki immediately after startup. This stack gives you the complete observability picture: metrics, logs, and alerting in one place.
