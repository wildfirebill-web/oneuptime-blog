# How to Build Grafana Dashboards for IPv4 Network Traffic Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Grafana, IPv4, Network Traffic, Dashboards, Prometheus, Visualization

Description: Build Grafana dashboards to visualize IPv4 network traffic metrics from Prometheus, including bandwidth graphs, packet rates, and per-host network utilization panels.

## Introduction

Network traffic dashboards in Grafana visualize bandwidth utilization, packet rates, and error counts per interface. Using Prometheus as the data source and Node Exporter metrics, you can build comprehensive network monitoring dashboards with minimal configuration.

## Dashboard JSON Panels

```json
// Import this dashboard via Grafana UI → Dashboards → Import
// Or use the provisioning approach with a JSON file

{
  "title": "IPv4 Network Traffic",
  "panels": [
    {
      "title": "Network Bandwidth (Mbps)",
      "type": "timeseries",
      "targets": [
        {
          "expr": "rate(node_network_receive_bytes_total{device!='lo',instance='$instance'}[5m]) * 8 / 1000000",
          "legendFormat": "Inbound - {{device}}"
        },
        {
          "expr": "rate(node_network_transmit_bytes_total{device!='lo',instance='$instance'}[5m]) * 8 / 1000000",
          "legendFormat": "Outbound - {{device}}"
        }
      ]
    }
  ]
}
```

## PromQL Queries for Dashboard Panels

```promql
# Panel 1: Inbound bandwidth per interface (Mbps)

rate(node_network_receive_bytes_total{device!~"lo|docker.*|veth.*",instance="$instance"}[5m]) * 8 / 1000000

# Panel 2: Outbound bandwidth per interface (Mbps)
rate(node_network_transmit_bytes_total{device!~"lo|docker.*|veth.*",instance="$instance"}[5m]) * 8 / 1000000

# Panel 3: Packet receive rate (pps)
rate(node_network_receive_packets_total{device!="lo",instance="$instance"}[5m])

# Panel 4: Packet drop rate
rate(node_network_receive_drop_total{device!="lo",instance="$instance"}[5m])

# Panel 5: Network error rate
rate(node_network_receive_errs_total{device!="lo",instance="$instance"}[5m]) +
rate(node_network_transmit_errs_total{device!="lo",instance="$instance"}[5m])

# Panel 6: Top 10 hosts by bandwidth (for fleet overview)
topk(10,
  sum by (instance) (
    rate(node_network_receive_bytes_total{device!="lo"}[5m])
  ) * 8 / 1000000
)
```

## Dashboard Variables

```json
// Template variable for instance selection
{
  "name": "instance",
  "type": "query",
  "query": "label_values(node_network_info, instance)",
  "refresh": 2,
  "multi": false
}

// Variable for device/interface selection
{
  "name": "device",
  "type": "query",
  "query": "label_values(node_network_info{instance='$instance'}, device)",
  "refresh": 2,
  "multi": true,
  "includeAll": true
}
```

## Provisioning Dashboards

```yaml
# /etc/grafana/provisioning/dashboards/default.yml

apiVersion: 1

providers:
  - name: 'Network Dashboards'
    orgId: 1
    folder: 'Infrastructure'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 60
    options:
      path: /etc/grafana/dashboards
```

```bash
# Place dashboard JSON files in:
# /etc/grafana/dashboards/network-traffic.json

# Grafana will automatically load them
sudo systemctl reload grafana-server
```

## Alert Rules in Dashboard

```yaml
# Alert panel example (PromQL):
# Alert when inbound bandwidth exceeds 800 Mbps

# Condition: WHEN avg() OF query(A, 5m, now) IS ABOVE 800
# Notification channel: email/slack/pagerduty
```

## Conclusion

Build Grafana network traffic dashboards using `node_network_receive/transmit_bytes_total` metrics from Node Exporter. Use `rate()` to convert cumulative counters to per-second values and multiply by 8 and divide by 1000000 for Mbps. Create template variables for instance and device to make dashboards reusable across all hosts. Provision dashboards via YAML files in `/etc/grafana/provisioning/dashboards/` for version-controlled dashboard management.
