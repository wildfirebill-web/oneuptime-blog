# How to Implement Monitoring Best Practices in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Monitoring, Prometheus, Grafana, Alerting, Observability

Description: Implement monitoring best practices in Rancher using the Prometheus stack, defining SLOs, configuring alerting, and establishing dashboards that cover cluster health, application metrics, and...

## Introduction

Monitoring in Rancher must cover multiple layers: the Rancher management plane, Kubernetes cluster infrastructure (nodes, etcd, control plane), and application workloads. The Rancher Monitoring app (based on kube-prometheus-stack) provides a complete starting point, but effective monitoring requires deliberate configuration of SLOs, alert thresholds, and dashboards.

## Step 1: Install Rancher Monitoring

```bash
# Enable from Rancher UI:

# Cluster > Apps > Charts > Monitoring
# Or install via Helm

helm repo add rancher-charts https://charts.rancher.io
helm install rancher-monitoring rancher-charts/rancher-monitoring \
  --namespace cattle-monitoring-system \
  --create-namespace \
  --set prometheus.prometheusSpec.retentionSize=50GB \
  --set prometheus.prometheusSpec.retention=30d \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.size=10Gi
```

## Step 2: Define Service Level Objectives (SLOs)

```yaml
# PrometheusRule defining SLO for API availability
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-slo
  namespace: production
spec:
  groups:
    - name: slo.availability
      rules:
        # 99.9% availability SLO (8.7 hours error budget/year)
        - record: job:slo_availability:ratio_rate5m
          expr: |
            sum(rate(http_requests_total{job="api",code!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="api"}[5m]))

        - alert: SLOAvailabilityBudgetBurn
          expr: job:slo_availability:ratio_rate5m < 0.999
          for: 1m
          annotations:
            summary: "API availability SLO breach"
          labels:
            severity: critical
```

## Step 3: Critical Alerts to Configure

```yaml
# PrometheusRule for infrastructure alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: infrastructure-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: nodes
      rules:
        - alert: NodeMemoryPressure
          expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.10
          for: 5m
          annotations:
            summary: "Node {{ $labels.node }} has less than 10% memory available"

        - alert: NodeDiskPressure
          expr: |
            (node_filesystem_avail_bytes{mountpoint="/"} /
             node_filesystem_size_bytes{mountpoint="/"}) < 0.15
          for: 10m
          annotations:
            summary: "Node {{ $labels.instance }} disk nearly full"

        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
          for: 5m
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash-looping"

        - alert: EtcdHighCommitDuration
          expr: histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])) > 0.25
          for: 5m
          annotations:
            summary: "etcd commit latency is high - storage performance issue"
```

## Step 4: Essential Grafana Dashboards

Import these dashboards from grafana.com:
- **15661**: Kubernetes Cluster Overview
- **15757**: Node Exporter Full
- **13332**: Kubernetes API Server
- **7249**: etcd by Prometheus
- **315**: Kubernetes cluster monitoring

```bash
# Import dashboard via Grafana API
curl -X POST http://grafana.company.com/api/dashboards/import \
  -H "Content-Type: application/json" \
  -d '{"dashboard": {"id": 15661}, "folderId": 0, "overwrite": true}'
```

## Step 5: Multi-Cluster Monitoring

```yaml
# Thanos or Mimir for aggregated multi-cluster metrics
# Configure Prometheus remote_write to central store

prometheusSpec:
  remoteWrite:
    - url: "https://mimir.monitoring.company.com/api/v1/push"
      headers:
        X-Scope-OrgID: "cluster-prod-east"
      basicAuth:
        username:
          name: remote-write-secret
          key: username
        password:
          name: remote-write-secret
          key: password
```

## Step 6: Configure Alertmanager Routing

```yaml
# alertmanager-config.yaml
global:
  slack_api_url: "https://hooks.slack.com/..."
  pagerduty_url: "https://events.pagerduty.com/..."

route:
  receiver: slack-default
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers: [severity="critical"]
      receiver: pagerduty-oncall
      continue: true
    - matchers: [severity="warning"]
      receiver: slack-warnings

receivers:
  - name: pagerduty-oncall
    pagerduty_configs:
      - service_key: "your-service-key"
  - name: slack-warnings
    slack_configs:
      - channel: "#k8s-alerts"
        send_resolved: true
```

## Monitoring Checklist

- Rancher Monitoring (kube-prometheus-stack) installed on all clusters
- SLOs defined for all customer-facing services
- Alerts for: node pressure, pod restarts, etcd latency, PVC usage
- Grafana dashboards for cluster, node, and application layers
- Alertmanager routing: critical → PagerDuty, warning → Slack
- Multi-cluster metrics aggregated via Thanos/Mimir
- Dashboards reviewed and updated quarterly
- Alert fatigue reviewed - suppress noise, fix root causes

## Conclusion

Effective monitoring in Rancher combines the technical stack (Prometheus, Grafana, Alertmanager) with deliberate SLO definitions and alert routing. Install Rancher Monitoring on every cluster, define SLOs for critical services, and ensure alerts reach the right people. For multi-cluster environments, aggregate metrics to a central Thanos or Mimir instance for cross-cluster visibility and long-term trend analysis.
