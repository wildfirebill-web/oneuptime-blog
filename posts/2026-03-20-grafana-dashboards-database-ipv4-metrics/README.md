# How to Set Up Grafana Dashboards for Database IPv4 Connection Metrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Grafana, Dashboard, PostgreSQL, MySQL, IPv4, Metrics, Monitoring, Visualization

Description: Learn how to build Grafana dashboards to visualize database connection metrics by IPv4 client address using Prometheus data sources.

---

Monitoring database connections by client IPv4 address helps identify connection leaks, unauthorized access, and capacity issues. This guide shows how to build Grafana dashboards for PostgreSQL and MySQL connection metrics.

## Prerequisites

- PostgreSQL Exporter or MySQL Exporter deployed and scraped by Prometheus.
- Grafana configured with Prometheus as a data source.

## PostgreSQL Connection Metrics Dashboard

### Key Metrics (PostgreSQL Exporter)

```promql
# Total active connections to PostgreSQL

pg_stat_activity_count{state="active"}

# Connections grouped by client IP address
pg_stat_activity_count by (client_addr)

# Idle connections (potential connection leaks)
pg_stat_activity_count{state="idle"}

# Connections waiting for locks
pg_stat_activity_count{wait_event_type!=""}
```

### Sample Grafana Panel Configuration

```json
// Grafana panel JSON (paste in "Code" view when adding a panel)
{
  "type": "timeseries",
  "title": "Active DB Connections by Client IPv4",
  "targets": [
    {
      "expr": "sum by (client_addr) (pg_stat_activity_count{state='active', client_addr!=''})",
      "legendFormat": "{{client_addr}}"
    }
  ],
  "options": {
    "tooltip": { "mode": "multi" }
  }
}
```

## MySQL Connection Metrics Dashboard

### Key Metrics (MySQL Exporter)

```promql
# Total MySQL connections
mysql_global_status_threads_connected

# Max connections ever reached (compare to max_connections limit)
mysql_global_status_max_used_connections

# Connections that were aborted (authentication failures, network issues)
mysql_global_status_aborted_connects
rate(mysql_global_status_aborted_connects[5m])
```

## Example Grafana Dashboard YAML (Provisioning)

```yaml
# /etc/grafana/provisioning/dashboards/database-connections.yaml
apiVersion: 1
providers:
  - name: database-connections
    folder: Databases
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

```json
// /var/lib/grafana/dashboards/db-connections.json
{
  "title": "Database Connection Monitor",
  "panels": [
    {
      "id": 1,
      "title": "PostgreSQL Active Connections",
      "type": "gauge",
      "targets": [{ "expr": "sum(pg_stat_activity_count{state='active'})" }],
      "options": { "reduceOptions": { "calcs": ["lastNotNull"] } }
    },
    {
      "id": 2,
      "title": "Connections by Client IP",
      "type": "table",
      "targets": [
        {
          "expr": "sum by (client_addr) (pg_stat_activity_count{client_addr!=''})",
          "instant": true,
          "legendFormat": "{{client_addr}}"
        }
      ]
    },
    {
      "id": 3,
      "title": "Connection Rate Over Time",
      "type": "timeseries",
      "targets": [
        { "expr": "rate(mysql_global_status_connections[5m])", "legendFormat": "Connections/sec" },
        { "expr": "rate(mysql_global_status_aborted_connects[5m])", "legendFormat": "Aborted/sec" }
      ]
    }
  ]
}
```

## Alerting on Too Many Connections

```yaml
# Grafana alert rule (via Grafana Alerting UI or provisioning)
# Alert when a single client IP has >50 connections
groups:
  - name: database-alerts
    rules:
      - alert: TooManyConnectionsFromIP
        expr: sum by (client_addr) (pg_stat_activity_count{client_addr!=""}) > 50
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High connection count from {{ $labels.client_addr }}"
          description: "Client {{ $labels.client_addr }} has {{ $value }} active connections"
```

## Key Takeaways

- `pg_stat_activity_count by (client_addr)` groups PostgreSQL connections by source IPv4 address.
- Use Grafana table panels for snapshot views of current connection counts per IP.
- Set up Grafana alerts on `pg_stat_activity_count` or `mysql_global_status_threads_connected` to catch connection spikes.
- Dashboard provisioning via YAML files makes dashboards version-controllable and reproducible.
