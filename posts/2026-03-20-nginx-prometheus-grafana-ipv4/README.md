# How to Monitor Nginx IPv4 Connections with Prometheus and Grafana (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, Prometheus, Grafana, IPv4, Monitoring, Metric

Description: Export Nginx connection and request metrics to Prometheus using nginx-prometheus-exporter, scrape with Prometheus, and build Grafana dashboards for connection monitoring.

## Introduction

Nginx exposes basic stats via the `ngx_http_stub_status_module`. The `nginx-prometheus-exporter` converts these into Prometheus metrics. For more detailed metrics (per-upstream, per-location), Nginx Plus or the third-party `nginx-module-vts` are required.

## Enabling nginx_stub_status

```nginx
# /etc/nginx/conf.d/stub_status.conf

server {
    # Bind to monitoring interface only
    listen 10.0.0.5:8080;
    server_name _;

    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        allow 10.0.0.0/24;    # Prometheus server network
        deny all;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx

# Test stub status

curl http://10.0.0.5:8080/nginx_status
# Output:
# Active connections: 42
# server accepts handled requests
#  1234 1234 5678
# Reading: 3 Writing: 10 Waiting: 29
```

## Installing nginx-prometheus-exporter

```bash
# Download nginx-prometheus-exporter
EXPORTER_VERSION="0.11.0"
wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v${EXPORTER_VERSION}/nginx-prometheus-exporter_${EXPORTER_VERSION}_linux_amd64.tar.gz
tar xzf nginx-prometheus-exporter_*.tar.gz
sudo cp nginx-prometheus-exporter /usr/local/bin/

# systemd service
cat > /etc/systemd/system/nginx-prometheus-exporter.service << 'EOF'
[Unit]
Description=Nginx Prometheus Exporter
After=network.target

[Service]
ExecStart=/usr/local/bin/nginx-prometheus-exporter \
  -nginx.scrape-uri=http://10.0.0.5:8080/nginx_status \
  -web.listen-address=10.0.0.5:9113

Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable --now nginx-prometheus-exporter
```

## Prometheus Scrape Configuration

```yaml
# /etc/prometheus/prometheus.yml

scrape_configs:
  - job_name: 'nginx'
    static_configs:
      - targets:
          - '10.0.0.1:9113'
          - '10.0.0.2:9113'
          - '10.0.0.3:9113'
        labels:
          role: web_server
    scrape_interval: 15s
```

## Key Metrics and PromQL Queries

```promql
# Active connections
nginx_connections_active

# Connection handling rate (per second)
rate(nginx_connections_accepted[5m])

# Handled vs accepted (dropped connections if != 1.0)
rate(nginx_connections_handled[5m]) / rate(nginx_connections_accepted[5m])

# Request rate per second
rate(nginx_http_requests_total[5m])

# Connections waiting for keepalive
nginx_connections_waiting

# Active connections per server
nginx_connections_active{instance=~"10\\.0\\.0\\..*:9113"}
```

## Grafana Dashboard Panels

```promql
# Panel 1: Active Connections (Stat panel)
nginx_connections_active

# Panel 2: Request Rate (Time series)
rate(nginx_http_requests_total[5m])

# Panel 3: Connection Status (Bar gauge)
# Reading / Writing / Waiting
nginx_connections_reading
nginx_connections_writing
nginx_connections_waiting

# Panel 4: Request Acceptance Rate
rate(nginx_connections_accepted[5m])
```

## Alerting Rules

```yaml
# /etc/prometheus/rules/nginx_alerts.yml

groups:
  - name: nginx
    rules:
      - alert: NginxDown
        expr: nginx_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Nginx is down on {{ $labels.instance }}"

      - alert: NginxHighConnections
        expr: nginx_connections_active > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High active connections on {{ $labels.instance }}: {{ $value }}"
```

## Conclusion

Monitor Nginx with `nginx-prometheus-exporter` which converts stub_status output to Prometheus format. Enable `stub_status` on a monitoring-only listen address, run the exporter pointing to that URL, and scrape it with Prometheus. Build Grafana panels for `nginx_connections_active`, `nginx_http_requests_total`, and the reading/writing/waiting connection breakdown. For per-upstream metrics, consider `nginx-module-vts` or Nginx Plus.
