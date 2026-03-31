# How to Create Grafana Dashboards for IPv6 Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Grafana, IPv6, Dashboard, Prometheus, Monitoring, Visualization

Description: A guide to building Grafana dashboards that visualize IPv6 traffic metrics from Prometheus, including throughput, packet rates, and error tracking.

Grafana dashboards make IPv6 traffic metrics actionable by providing visual context for throughput trends, error spikes, and comparative IPv4/IPv6 analysis. This guide walks through building a practical IPv6 network dashboard.

## Dashboard Architecture

```mermaid
flowchart LR
    NE[Node Exporter] --> P[Prometheus]
    BE[Blackbox Exporter] --> P
    P --> G[Grafana Dashboard]
    G --> R[IPv6 Traffic Row]
    G --> S[IPv6 Errors Row]
    G --> U[IPv6 Uptime Row]
```

## Step 1: Create the Dashboard via Grafana API

```bash
# Create dashboard via Grafana API

curl -X POST http://admin:password@localhost:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d @ipv6-dashboard.json
```

## Step 2: Key Panels and Queries

### Panel 1: IPv6 Inbound Traffic Rate

```json
{
  "title": "IPv6 Inbound Traffic (bytes/sec)",
  "type": "timeseries",
  "targets": [
    {
      "expr": "rate(node_netstat_Ip6_InReceives{instance=~\"$instance\"}[5m])",
      "legendFormat": "{{instance}} - IPv6 In"
    },
    {
      "expr": "rate(node_netstat_Ip4_InReceives{instance=~\"$instance\"}[5m])",
      "legendFormat": "{{instance}} - IPv4 In"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "pps"
    }
  }
}
```

### Panel 2: IPv6 vs IPv4 Traffic Share (Pie Chart)

```promql
# PromQL for IPv6 proportion of total traffic
sum(rate(node_netstat_Ip6_InReceives[5m]))
/
(sum(rate(node_netstat_Ip6_InReceives[5m])) + sum(rate(node_netstat_InReceives[5m])))
* 100
```

### Panel 3: ICMPv6 Message Types

```promql
# ICMPv6 inbound message rate
rate(node_netstat_Icmp6_InMsgs[5m])

# ICMPv6 error rate (should be low)
rate(node_netstat_Icmp6_InErrors[5m])
```

### Panel 4: IPv6 Uptime Probe (from Blackbox Exporter)

```promql
# IPv6 endpoint availability
probe_success{job="blackbox-http-ipv6"}

# IPv6 HTTP response latency
probe_duration_seconds{job="blackbox-http-ipv6"}
```

## Step 3: Complete Dashboard JSON (Excerpt)

```json
{
  "dashboard": {
    "title": "IPv6 Network Monitoring",
    "tags": ["ipv6", "networking"],
    "timezone": "browser",
    "templating": {
      "list": [
        {
          "name": "instance",
          "type": "query",
          "query": "label_values(node_netstat_Ip6_InReceives, instance)",
          "multi": true,
          "includeAll": true
        }
      ]
    },
    "panels": [
      {
        "title": "IPv6 Packet Receive Rate",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "rate(node_netstat_Ip6_InReceives{instance=~\"$instance\"}[5m])",
            "legendFormat": "{{instance}}"
          }
        ]
      },
      {
        "title": "IPv6 No-Route Errors",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0},
        "targets": [
          {
            "expr": "rate(node_netstat_Ip6_OutNoRoutes{instance=~\"$instance\"}[5m])",
            "legendFormat": "{{instance}} no-route"
          }
        ],
        "alert": {
          "conditions": [{"evaluator": {"type": "gt", "params": [5]}}]
        }
      }
    ]
  }
}
```

## Step 4: Import the Community IPv6 Dashboard

The Grafana dashboard repository includes community-contributed IPv6 dashboards:

```bash
# Import dashboard ID 1860 (Node Exporter Full) which includes IPv6 panels
curl -X POST http://admin:password@localhost:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  -d '{"dashboardId": 1860, "overwrite": true, "folderId": 0}'
```

## Step 5: Create an IPv6 Adoption Tracking Panel

```promql
# Track what percentage of your servers have IPv6 addresses
count(node_netstat_Ip6_InReceives > 0)
/
count(node_netstat_InReceives)
* 100
```

Grafana IPv6 dashboards provide immediate visual insight into the health and adoption rate of IPv6 across your infrastructure, making it easy to spot routing issues, traffic anomalies, and service unavailability.
