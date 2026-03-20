# How to Monitor Multi-Cluster Health from Rancher Dashboard - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Monitoring, Dashboards, Multi-Cluster

Description: Build a comprehensive multi-cluster health monitoring setup in Rancher using the built-in dashboard, Prometheus, Grafana, and custom alerting to maintain visibility across all clusters.

## Introduction

With Rancher managing dozens or hundreds of clusters, having a reliable health dashboard is essential for on-call engineers and platform teams. Rancher provides a built-in cluster summary view, and when combined with Prometheus, Grafana, and alerting, it becomes a powerful multi-cluster observability platform. This guide builds a complete, production-ready multi-cluster health monitoring solution.

## Step 1: Use Rancher's Built-In Dashboard

The Rancher home dashboard provides an at-a-glance view of all clusters:

1. Navigate to **☰ → Home**.
2. The **Cluster Summary** shows:
   - Cluster status (Active/Unavailable)
   - Kubernetes version
   - Node count and health
   - CPU and memory utilization
3. Click any cluster name to drill into its specific metrics.

```bash
# Check cluster states via API

curl -sk \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "https://rancher.example.com/v3/clusters?limit=-1" \
  | jq '.data[] | {name: .name, state: .state, version: .version}'
```

## Step 2: Install Rancher Monitoring on All Clusters

```bash
# Install via Fleet - apply to all clusters
cat << 'EOF' | kubectl apply -f -
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: monitoring-stack
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/cluster-config
  branch: main
  paths:
    - monitoring/
  targets:
    - clusterSelector: {}   # All clusters
EOF
```

```yaml
# monitoring/helmchart.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: rancher-monitoring
  namespace: kube-system
spec:
  repo: https://charts.rancher.io
  chart: rancher-monitoring
  targetNamespace: cattle-monitoring-system
  createNamespace: true
  valuesContent: |
    prometheus:
      prometheusSpec:
        retention: 7d
        storageSpec:
          volumeClaimTemplate:
            spec:
              resources:
                requests:
                  storage: 20Gi
    grafana:
      enabled: true
      adminPassword: ChangeMeNow!
```

## Step 3: Configure Centralized Metrics Collection

```yaml
# Configure Thanos for cross-cluster metrics (on the management cluster)
# prometheus-federation.yaml

# Additional scrape config for the central Prometheus
- job_name: 'cluster-federation'
  honor_labels: true
  metrics_path: '/federate'
  params:
    match[]:
      # Essential cluster health metrics only
      - 'up'
      - 'kube_node_status_condition'
      - 'kube_pod_status_phase'
      - 'container_cpu_usage_seconds_total'
      - 'container_memory_working_set_bytes'
      - 'kube_deployment_status_replicas_unavailable'
      - 'kube_daemonset_status_number_unavailable'
      - 'etcd_server_has_leader'
  relabel_configs:
    - source_labels: [__address__]
      target_label: cluster
      regex: '(.+)\.svc\.cluster\.local.*'
      replacement: '$1'
  static_configs:
    - targets:
        - 'prometheus.cattle-monitoring-system.cluster-1.svc:9090'
        - 'prometheus.cattle-monitoring-system.cluster-2.svc:9090'
      labels:
        datacenter: us-east
```

## Step 4: Create a Multi-Cluster Health Dashboard

```json
{
  "title": "Multi-Cluster Health Overview",
  "uid": "multi-cluster-health",
  "panels": [
    {
      "title": "Cluster Status",
      "type": "stat",
      "gridPos": {"h": 4, "w": 24, "x": 0, "y": 0},
      "targets": [{
        "expr": "count(up{job='kube-state-metrics'}) by (cluster)",
        "legendFormat": "{{cluster}}"
      }],
      "options": {"reduceOptions": {"calcs": ["lastNotNull"]}}
    },
    {
      "title": "Nodes Not Ready",
      "type": "table",
      "gridPos": {"h": 8, "w": 12, "x": 0, "y": 4},
      "targets": [{
        "expr": "kube_node_status_condition{condition='Ready',status='false'} == 1",
        "legendFormat": "{{cluster}} / {{node}}"
      }]
    },
    {
      "title": "CPU Usage % by Cluster",
      "type": "timeseries",
      "gridPos": {"h": 8, "w": 12, "x": 12, "y": 4},
      "targets": [{
        "expr": "100 * sum(rate(container_cpu_usage_seconds_total{container!=''}[5m])) by (cluster) / sum(kube_node_status_capacity{resource='cpu'}) by (cluster)",
        "legendFormat": "{{cluster}}"
      }]
    },
    {
      "title": "Failed Pods by Cluster",
      "type": "timeseries",
      "gridPos": {"h": 8, "w": 12, "x": 0, "y": 12},
      "targets": [{
        "expr": "count(kube_pod_status_phase{phase=~'Failed|Unknown'}) by (cluster, namespace)",
        "legendFormat": "{{cluster}} / {{namespace}}"
      }]
    },
    {
      "title": "etcd Health",
      "type": "stat",
      "gridPos": {"h": 4, "w": 12, "x": 12, "y": 12},
      "targets": [{
        "expr": "etcd_server_has_leader by (cluster)",
        "legendFormat": "{{cluster}}"
      }],
      "options": {
        "colorMode": "background",
        "thresholds": {"steps": [{"color": "red", "value": 0}, {"color": "green", "value": 1}]}
      }
    }
  ]
}
```

## Step 5: Configure Multi-Cluster Alerts

```yaml
# PrometheusRule for critical multi-cluster alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: multi-cluster-critical-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: cluster-health
      interval: 30s
      rules:
        # Alert if any cluster is down (no metrics from it)
        - alert: ClusterDown
          expr: absent(up{job="kube-state-metrics"})
          for: 2m
          labels:
            severity: critical
            page: "true"
          annotations:
            summary: "Cluster {{ $labels.cluster }} is unreachable"
            description: "No metrics have been received from {{ $labels.cluster }} for 2 minutes"

        # Alert on high percentage of NotReady nodes
        - alert: ClusterNodeAvailabilityLow
          expr: |
            (count(kube_node_status_condition{condition="Ready",status="true"}) by (cluster)
            / count(kube_node_info) by (cluster)) < 0.8
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Less than 80% of nodes Ready in {{ $labels.cluster }}"

        # Alert if etcd loses leader
        - alert: EtcdHasNoLeader
          expr: etcd_server_has_leader == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "etcd in {{ $labels.cluster }} has no leader"

        # Alert on deployment unavailability
        - alert: DeploymentReplicasUnavailable
          expr: kube_deployment_status_replicas_unavailable > 0
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} in {{ $labels.cluster }} has unavailable replicas"
```

## Step 6: Configure Alert Routing

```yaml
# Alertmanager configuration for multi-cluster alerting
route:
  group_by: ['cluster', 'alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  routes:
    # Critical alerts go to PagerDuty
    - match:
        severity: critical
      receiver: pagerduty-critical

    # Warning alerts go to Slack
    - match:
        severity: warning
      receiver: slack-warnings

receivers:
  - name: pagerduty-critical
    pagerduty_configs:
      - routing_key: <pagerduty-integration-key>
        description: '{{ .GroupLabels.cluster }}: {{ .CommonAnnotations.summary }}'

  - name: slack-warnings
    slack_configs:
      - api_url: <slack-webhook-url>
        channel: '#k8s-alerts'
        text: '{{ range .Alerts }}*{{ .Annotations.summary }}*\n{{ .Annotations.description }}\n{{ end }}'
```

## Step 7: Automate Health Reports

```bash
#!/usr/bin/env bash
# cluster-health-report.sh - Daily health report across all clusters

RANCHER_URL="https://rancher.example.com"
TOKEN="${RANCHER_TOKEN}"

echo "=== Multi-Cluster Health Report: $(date) ==="

# Get all clusters and their states
curl -sk \
  -H "Authorization: Bearer ${TOKEN}" \
  "${RANCHER_URL}/v3/clusters?limit=-1" \
  | jq -r '.data[] | "\(.name)\t\(.state)\t\(.version)\tNodes: \(.nodeCount)"' \
  | column -t

echo ""
echo "=== Clusters with Issues ==="
curl -sk \
  -H "Authorization: Bearer ${TOKEN}" \
  "${RANCHER_URL}/v3/clusters?limit=-1" \
  | jq -r '.data[] | select(.state != "active") | "\(.name): \(.state) - \(.conditions[-1].message // "no message")"'
```

## Conclusion

A comprehensive multi-cluster health monitoring setup in Rancher combines the built-in dashboard for quick overviews, Prometheus and Grafana for deep metric analysis, and Alertmanager for proactive notifications. By federating metrics to a central Prometheus and creating cross-cluster Grafana dashboards, your platform team gains unified visibility into the health of your entire Kubernetes estate, enabling faster incident response and proactive capacity planning.
