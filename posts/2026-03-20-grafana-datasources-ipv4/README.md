# How to Configure Grafana Data Sources with IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Grafana, IPv4, Data Source, Prometheus, InfluxDB, Configuration

Description: Configure Grafana data sources to connect to Prometheus, InfluxDB, and other backends via IPv4 addresses, using both the UI and provisioning files.

## Introduction

Grafana data sources connect to time-series databases and metric backends. Each data source is configured with a URL (using IPv4 or hostname) for the backend. Proper data source configuration is the foundation of all Grafana dashboards.

## Prometheus Data Source

```yaml
# /etc/grafana/provisioning/datasources/prometheus.yml

# Provisioning file - automatically loaded at startup

apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://10.0.0.5:9090
    isDefault: true
    editable: false
    jsonData:
      timeInterval: "15s"
      httpMethod: POST
      queryTimeout: "60s"
```

## Multiple Prometheus Sources

```yaml
# /etc/grafana/provisioning/datasources/all_datasources.yml

apiVersion: 1

datasources:
  - name: Prometheus-Production
    type: prometheus
    access: proxy
    url: http://10.0.1.5:9090
    isDefault: true
    jsonData:
      customQueryParameters: "cluster=production"

  - name: Prometheus-Staging
    type: prometheus
    access: proxy
    url: http://10.0.2.5:9090
    jsonData:
      customQueryParameters: "cluster=staging"

  - name: InfluxDB
    type: influxdb
    access: proxy
    url: http://10.0.0.20:8086
    jsonData:
      dbName: metrics
      httpMode: GET

  - name: Loki
    type: loki
    access: proxy
    url: http://10.0.0.25:3100
```

## MySQL/PostgreSQL Data Sources

```yaml
datasources:
  - name: MySQL-Production
    type: mysql
    url: 10.0.0.10:3306
    user: grafana_ro
    secureJsonData:
      password: ReadOnlyPass
    jsonData:
      database: appdb
      maxOpenConns: 10

  - name: PostgreSQL-Production
    type: postgres
    url: 10.0.0.11:5432
    user: grafana_ro
    secureJsonData:
      password: ReadOnlyPass
    jsonData:
      database: appdb
      sslmode: require
```

## Alertmanager Data Source

```yaml
datasources:
  - name: Alertmanager
    type: alertmanager
    access: proxy
    url: http://10.0.0.5:9093
    jsonData:
      implementation: prometheus
```

## Applying Provisioning

```bash
# Apply changes by restarting Grafana
sudo systemctl restart grafana-server

# Verify data sources loaded
curl -s -u admin:password \
  http://10.0.0.5:3000/api/datasources | \
  python3 -c "import sys,json; [print(d['name'],d['url']) for d in json.load(sys.stdin)]"

# Test a specific data source
curl -s -u admin:password \
  http://10.0.0.5:3000/api/datasources/name/Prometheus | python3 -m json.tool
```

## Via Grafana UI

```bash
# In the Grafana web interface:
# Settings (gear icon) → Data Sources → Add data source
# Select type, enter URL (http://10.0.0.5:9090), click "Save & Test"

# "Data source connected and labels found" = working
# "HTTP Error Bad Gateway" = Grafana can't reach the backend IP
```

## Conclusion

Configure Grafana data sources with IPv4 URLs using provisioning YAML files in `/etc/grafana/provisioning/datasources/`. Provisioned data sources are automatically created or updated at startup. Use `access: proxy` so Grafana server makes the connection (not the user's browser). Verify data sources work with the "Save & Test" button in the UI or the datasources API.
