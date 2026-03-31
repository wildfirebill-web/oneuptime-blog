# How to Deploy Grafana via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Grafana, Monitoring, Dashboard, Docker

Description: Learn how to deploy Grafana via Portainer with persistent storage, pre-configured data sources, and dashboard provisioning for a production-ready monitoring setup.

## Grafana via Portainer Stack

**Stacks → Add Stack → grafana**

```yaml
version: "3.8"

services:
  grafana:
    image: grafana/grafana:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      # Admin credentials
      - GF_SECURITY_ADMIN_USER=${GRAFANA_ADMIN_USER}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
      # Disable registration
      - GF_USERS_ALLOW_SIGN_UP=false
      # Server settings
      - GF_SERVER_ROOT_URL=https://grafana.yourdomain.com
      # SMTP for alerts (optional)
      - GF_SMTP_ENABLED=true
      - GF_SMTP_HOST=${SMTP_HOST}
      - GF_SMTP_USER=${SMTP_USER}
      - GF_SMTP_PASSWORD=${SMTP_PASSWORD}
      - GF_SMTP_FROM_ADDRESS=grafana@yourdomain.com
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana-provisioning:/etc/grafana/provisioning:ro
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:3000/api/health | grep -q ok"]
      interval: 10s
      retries: 5

volumes:
  grafana_data:
```

## Environment Variables

```text
GRAFANA_ADMIN_USER = admin
GRAFANA_ADMIN_PASSWORD = secure-grafana-password
SMTP_HOST = smtp.gmail.com:587
SMTP_USER = alerts@yourdomain.com
SMTP_PASSWORD = smtp-app-password
```

## Provisioning Data Sources

Create `grafana-provisioning/datasources/datasources.yml`:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100

  - name: InfluxDB
    type: influxdb
    access: proxy
    url: http://influxdb:8086
    jsonData:
      version: Flux
      organization: myorg
      defaultBucket: metrics
    secureJsonData:
      token: ${INFLUXDB_TOKEN}
```

## Provisioning Dashboards

Create `grafana-provisioning/dashboards/dashboards.yml`:

```yaml
apiVersion: 1

providers:
  - name: default
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 60
    options:
      path: /etc/grafana/provisioning/dashboards
```

Place dashboard JSON files in the same directory. Download community dashboards:

```bash
# Download Node Exporter Full dashboard

curl -s "https://grafana.com/api/dashboards/1860/revisions/36/download" \
  > grafana-provisioning/dashboards/node-exporter.json
```

## Grafana with Prometheus and Alertmanager

```yaml
services:
  grafana:
    # ... (above)

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
```

## Grafana Plugin Installation

```yaml
environment:
  # Install plugins at startup
  - GF_INSTALL_PLUGINS=grafana-piechart-panel,grafana-worldmap-panel,grafana-clock-panel
```

## Accessing Grafana

Open `http://server:3000`.

Login with admin / GRAFANA_ADMIN_PASSWORD.

## Creating Your First Alert

1. **Alerting → Alert Rules → New alert rule**
2. Set query: `avg(rate(http_requests_total{status!="200"}[5m])) > 0.1`
3. Set evaluation: every 1m for 5m
4. Set notification policy to route to your contact points (email/Slack)

## Conclusion

Grafana via Portainer with provisioning files delivers a pre-configured monitoring dashboard platform that's reproducible across deployments. The provisioning approach means data sources and dashboards are automatically configured on startup - no manual setup required when recreating the stack.
