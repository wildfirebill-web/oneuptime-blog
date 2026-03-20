# How to Monitor Rancher Server Resource Usage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Monitoring, Prometheus, Grafana, Resource Usage

Description: Set up comprehensive monitoring for Rancher server resource usage with Prometheus metrics, Grafana dashboards, and alerting for proactive capacity management.

## Introduction

Monitoring Rancher server resource usage is essential for maintaining reliable cluster management at any scale. This guide covers setting up Prometheus metrics collection for Rancher, creating Grafana dashboards, and configuring alerts to detect capacity issues before they impact operations.

## Prerequisites

- Rancher-monitoring installed (Prometheus Operator)
- Grafana accessible
- Cluster admin access to Rancher's local cluster

## Step 1: Enable Rancher Server Metrics

```bash
# Rancher exposes Prometheus metrics at /metrics
# Verify metrics are accessible
curl -sk \
  -H "Authorization: Bearer $TOKEN" \
  "https://rancher.example.com/metrics" | head -50

# Check if ServiceMonitor exists
kubectl get servicemonitor -n cattle-system
```

## Step 2: Create ServiceMonitor for Rancher

```yaml
# rancher-servicemonitor.yaml - Scrape Rancher metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rancher-server
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  namespaceSelector:
    matchNames:
      - cattle-system
  selector:
    matchLabels:
      app: rancher
  endpoints:
    - port: https-80
      path: /metrics
      scheme: https
      tlsConfig:
        insecureSkipVerify: true
      bearerTokenSecret:
        name: rancher-monitoring-token
        key: token
      interval: 30s
```

## Step 3: Key Metrics to Monitor

```promql
# Rancher API request rate by endpoint
rate(rancher_api_request_total[5m])

# API error rate
sum(rate(rancher_api_request_total{code=~"5.."}[5m])) /
sum(rate(rancher_api_request_total[5m]))

# API p99 latency
histogram_quantile(0.99,
  rate(rancher_api_request_duration_seconds_bucket[5m])
)

# Active websocket connections (cluster agents)
rancher_cluster_count

# Number of healthy clusters
sum(rancher_cluster_ready == 1)

# Number of unhealthy clusters
sum(rancher_cluster_ready == 0)

# Rancher pod memory usage
container_memory_working_set_bytes{
  namespace="cattle-system",
  container="rancher"
}

# Rancher pod CPU usage
rate(container_cpu_usage_seconds_total{
  namespace="cattle-system",
  container="rancher"
}[5m])
```

## Step 4: Create Grafana Dashboard

```json
{
  "title": "Rancher Server Overview",
  "panels": [
    {
      "title": "Total Managed Clusters",
      "type": "stat",
      "targets": [{"expr": "rancher_cluster_count"}]
    },
    {
      "title": "Healthy Clusters",
      "type": "stat",
      "targets": [{"expr": "sum(rancher_cluster_ready == 1)"}]
    },
    {
      "title": "API Request Rate",
      "type": "graph",
      "targets": [{"expr": "sum(rate(rancher_api_request_total[5m])) by (code)"}]
    },
    {
      "title": "API p99 Latency",
      "type": "graph",
      "targets": [{"expr": "histogram_quantile(0.99, rate(rancher_api_request_duration_seconds_bucket[5m]))"}]
    },
    {
      "title": "Rancher Memory Usage",
      "type": "graph",
      "targets": [{"expr": "container_memory_working_set_bytes{namespace='cattle-system',container='rancher'}"}]
    }
  ]
}
```

## Step 5: Configure Critical Alerts

```yaml
# rancher-server-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: rancher-server-alerts
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  groups:
    - name: rancher-server
      rules:
        # Cluster agent disconnected
        - alert: RancherClusterDisconnected
          expr: rancher_cluster_ready == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Rancher cluster {{ $labels.cluster_id }} is disconnected"
            description: "Cluster has been unresponsive for 5+ minutes"

        # High memory usage
        - alert: RancherHighMemory
          expr: |
            container_memory_working_set_bytes{
              namespace="cattle-system",container="rancher"
            } / 1024 / 1024 / 1024 > 12
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Rancher server memory > 12GB"

        # API error rate high
        - alert: RancherAPIErrors
          expr: |
            sum(rate(rancher_api_request_total{code=~"5.."}[5m])) /
            sum(rate(rancher_api_request_total[5m])) > 0.1
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Rancher API error rate > 10%"

        # etcd database approaching quota
        - alert: RancherEtcdQuota
          expr: |
            etcd_mvcc_db_total_size_in_bytes /
            etcd_server_quota_backend_bytes > 0.8
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Rancher etcd database >80% of quota"
```

## Step 6: Monitor Fleet GitOps Performance

```promql
# Fleet bundle deployment success rate
sum(fleet_bundle_state{state="Ready"}) /
sum(fleet_bundle_state) * 100

# Fleet bundle errors
sum(fleet_bundle_state{state="Errored"})

# Clusters out of sync
sum(fleet_cluster_state{state="NotReady"})
```

## Step 7: Set Up Capacity Planning Metrics

```bash
# Create a recording rule for capacity trends
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: rancher-capacity
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: rancher-capacity
      interval: 5m
      rules:
        - record: rancher:cluster_count:total
          expr: rancher_cluster_count

        - record: rancher:api_request_rate:5m
          expr: sum(rate(rancher_api_request_total[5m]))

        - record: rancher:memory_usage_bytes
          expr: |
            sum(container_memory_working_set_bytes{
              namespace="cattle-system",
              container="rancher"
            })
EOF
```

## Conclusion

Comprehensive Rancher server monitoring enables proactive capacity management and rapid incident response. The key metrics to track are cluster connectivity (healthy vs disconnected), API performance (latency and error rates), resource consumption (CPU and memory), and etcd database health. Setting up alerts for cluster disconnection, high memory usage, and etcd quota utilization ensures that capacity issues are addressed before they impact the management plane's reliability.
