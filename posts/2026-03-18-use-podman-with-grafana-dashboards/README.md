# How to Use Podman with Grafana for Dashboards

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Grafana, Dashboards, Monitoring, Visualization

Description: Learn how to deploy Grafana in Podman containers to create rich monitoring dashboards for your containerized applications and infrastructure.

---

> Grafana running in Podman containers transforms raw metrics into actionable dashboards, giving you visual insight into the health and performance of your containerized infrastructure.

Grafana is the leading open-source platform for monitoring visualization. It connects to data sources like Prometheus, Loki, and InfluxDB to create interactive dashboards that display metrics, logs, and traces in a unified interface. Running Grafana in a Podman container makes deployment simple and consistent, while connecting it to other containerized monitoring tools creates a complete observability stack.

---

## Deploying Grafana with Podman

Start a basic Grafana instance:

```bash
podman volume create grafana-data

podman run -d \
  --name grafana \
  --restart always \
  -p 3000:3000 \
  -v grafana-data:/var/lib/grafana:Z \
  -e GF_SECURITY_ADMIN_USER=admin \
  -e GF_SECURITY_ADMIN_PASSWORD=admin \
  grafana/grafana:latest
```

Access Grafana at `http://localhost:3000` and log in with the credentials you set.

## Full Monitoring Stack

Deploy Grafana alongside Prometheus and other monitoring tools:

```yaml
# monitoring-stack.yml
version: "3"
services:
  prometheus:
    image: prom/prometheus:latest
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus

  grafana:
    image: grafana/grafana:latest
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: "${GRAFANA_PASSWORD}"
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_SERVER_ROOT_URL: "http://localhost:3000"
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:latest
    restart: always
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'

volumes:
  prometheus-data:
  grafana-data:
```

```bash
GRAFANA_PASSWORD=securepass podman-compose -f monitoring-stack.yml up -d
```

## Provisioning Data Sources

Automatically configure data sources using Grafana's provisioning system:

```yaml
# grafana/provisioning/datasources/datasources.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
```

## Provisioning Dashboards

Automatically load dashboards on startup:

```yaml
# grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1

providers:
  - name: default
    orgId: 1
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

## Creating a Container Monitoring Dashboard

Create a dashboard JSON file for container monitoring:

```json
{
  "dashboard": {
    "title": "Container Monitoring",
    "tags": ["podman", "containers"],
    "timezone": "browser",
    "panels": [
      {
        "title": "Container CPU Usage",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "rate(container_cpu_usage_seconds_total[5m]) * 100",
            "legendFormat": "{{name}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "custom": {
              "drawStyle": "line",
              "fillOpacity": 20
            }
          }
        }
      },
      {
        "title": "Container Memory Usage",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0},
        "targets": [
          {
            "expr": "container_memory_usage_bytes",
            "legendFormat": "{{name}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "bytes"
          }
        }
      },
      {
        "title": "Container Network I/O",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8},
        "targets": [
          {
            "expr": "rate(container_network_receive_bytes_total[5m])",
            "legendFormat": "{{name}} - rx"
          },
          {
            "expr": "rate(container_network_transmit_bytes_total[5m])",
            "legendFormat": "{{name}} - tx"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "Bps"
          }
        }
      },
      {
        "title": "Running Containers",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 12, "y": 8},
        "targets": [
          {
            "expr": "count(container_last_seen)",
            "legendFormat": "Total"
          }
        ]
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "10s"
  }
}
```

Save this as `grafana/dashboards/container-monitoring.json`.

## Application Performance Dashboard

Create a dashboard for application metrics:

```json
{
  "dashboard": {
    "title": "Application Performance",
    "panels": [
      {
        "title": "Request Rate",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (endpoint)",
            "legendFormat": "{{endpoint}}"
          }
        ],
        "fieldConfig": {
          "defaults": {"unit": "reqps"}
        }
      },
      {
        "title": "Response Time (p95)",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0},
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint))",
            "legendFormat": "{{endpoint}}"
          }
        ],
        "fieldConfig": {
          "defaults": {"unit": "s"}
        }
      },
      {
        "title": "Error Rate",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8},
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100",
            "legendFormat": "Error %"
          }
        ],
        "fieldConfig": {
          "defaults": {"unit": "percent"}
        }
      }
    ]
  }
}
```

## Grafana Configuration Options

Configure Grafana through environment variables:

```bash
podman run -d \
  --name grafana \
  -p 3000:3000 \
  -v grafana-data:/var/lib/grafana:Z \
  -e GF_SECURITY_ADMIN_USER=admin \
  -e GF_SECURITY_ADMIN_PASSWORD=strongpassword \
  -e GF_USERS_ALLOW_SIGN_UP=false \
  -e GF_AUTH_ANONYMOUS_ENABLED=false \
  -e GF_SERVER_ROOT_URL=https://grafana.example.com \
  -e GF_SMTP_ENABLED=true \
  -e GF_SMTP_HOST=smtp.example.com:587 \
  -e GF_SMTP_USER=alerts@example.com \
  -e GF_SMTP_PASSWORD=smtppassword \
  -e GF_ALERTING_ENABLED=true \
  grafana/grafana:latest
```

## Backup and Restore

Back up Grafana data and dashboards:

```bash
#!/bin/bash
# backup-grafana.sh

BACKUP_DIR="/backups/grafana/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

# Export dashboards via API
GRAFANA_URL="http://localhost:3000"
API_KEY="your-api-key"

# Get all dashboard UIDs
curl -s -H "Authorization: Bearer $API_KEY" \
  "$GRAFANA_URL/api/search?type=dash-db" | \
  jq -r '.[].uid' | while read uid; do
    echo "Exporting dashboard: $uid"
    curl -s -H "Authorization: Bearer $API_KEY" \
      "$GRAFANA_URL/api/dashboards/uid/$uid" \
      > "$BACKUP_DIR/dashboard-$uid.json"
done

# Back up the Grafana volume
podman volume export grafana-data > "$BACKUP_DIR/grafana-volume.tar"

echo "Backup saved to $BACKUP_DIR"
```

Restore from backup:

```bash
# Restore the volume
podman volume create grafana-data
podman volume import grafana-data < /backups/grafana/20240101/grafana-volume.tar

# Restart Grafana
podman restart grafana
```

## Embedding Dashboards

Configure Grafana to allow embedding dashboards in other applications:

```bash
podman run -d \
  --name grafana \
  -p 3000:3000 \
  -v grafana-data:/var/lib/grafana:Z \
  -e GF_SECURITY_ALLOW_EMBEDDING=true \
  -e GF_AUTH_ANONYMOUS_ENABLED=true \
  -e GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer \
  grafana/grafana:latest
```

Then embed dashboards in your application using iframes or the Grafana embedding API.

## Conclusion

Grafana running in Podman containers provides powerful visualization capabilities for your monitoring data. The combination of Grafana's dashboard builder with Prometheus metrics gives you comprehensive insight into both infrastructure and application performance. Provisioning support means dashboards and data sources can be version-controlled and automatically deployed, making your monitoring infrastructure as reproducible as your application code. Whether you need simple resource monitoring or complex multi-service dashboards, Grafana and Podman together deliver an accessible and maintainable solution.
