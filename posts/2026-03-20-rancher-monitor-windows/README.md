# How to Monitor Windows Nodes in Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Windows, Monitoring, Prometheus, Node Exporter

Description: Set up monitoring for Windows nodes in Rancher using Windows Exporter, Prometheus, and Grafana dashboards for comprehensive Windows node observability.

## Introduction

Monitoring Windows nodes in Kubernetes requires Windows-specific exporters since the standard Linux Node Exporter doesn't run on Windows. The Windows Exporter (formerly wmi_exporter) collects Windows performance metrics for Prometheus. This guide covers deploying Windows Exporter as a DaemonSet and setting up Grafana dashboards.

## Prerequisites

- Rancher cluster with Windows worker nodes
- Rancher Monitoring (Prometheus Operator) installed
- kubectl access to the cluster

## Step 1: Deploy Windows Exporter DaemonSet

```yaml
# windows-exporter-daemonset.yaml - Deploy Windows Exporter on Windows nodes

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: windows-exporter
  namespace: cattle-monitoring-system
  labels:
    app: windows-exporter
spec:
  selector:
    matchLabels:
      app: windows-exporter
  template:
    metadata:
      labels:
        app: windows-exporter
    spec:
      # Only schedule on Windows nodes
      nodeSelector:
        kubernetes.io/os: windows

      tolerations:
        - key: os
          value: windows
          effect: NoSchedule

      containers:
        - name: windows-exporter
          image: ghcr.io/prometheus-community/windows-exporter:latest
          args:
            - --collectors.enabled=cpu,cs,logical_disk,memory,net,os,system,container,service,process
            - --telemetry.path=/metrics
          ports:
            - name: metrics
              containerPort: 9182
              protocol: TCP
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          securityContext:
            windowsOptions:
              runAsUserName: "LocalSystem"
```

## Step 2: Create Service for Windows Exporter

```yaml
# windows-exporter-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: windows-exporter
  namespace: cattle-monitoring-system
  labels:
    app: windows-exporter
spec:
  selector:
    app: windows-exporter
  ports:
    - name: metrics
      port: 9182
      targetPort: 9182
      protocol: TCP
  clusterIP: None  # Headless service for PodMonitor
```

## Step 3: Configure PodMonitor for Scraping

```yaml
# windows-exporter-podmonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: windows-exporter
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  namespaceSelector:
    matchNames:
      - cattle-monitoring-system
  selector:
    matchLabels:
      app: windows-exporter
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
      # Filter to Windows nodes only
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_node_name]
          targetLabel: node
        - sourceLabels: [__meta_kubernetes_pod_node_name]
          targetLabel: instance
```

## Step 4: Create Windows-Specific Alerts

```yaml
# windows-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: windows-node-alerts
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  groups:
    - name: windows.nodes
      rules:
        # High CPU usage
        - alert: WindowsHighCPU
          expr: |
            100 - (avg by (instance) (rate(windows_cpu_time_total{mode="idle"}[5m])) * 100) > 90
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Windows node {{ $labels.instance }} CPU > 90%"

        # Low memory
        - alert: WindowsLowMemory
          expr: |
            windows_os_physical_memory_free_bytes /
            windows_cs_physical_memory_bytes * 100 < 10
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Windows node {{ $labels.instance }} memory < 10% free"

        # Disk space low
        - alert: WindowsLowDiskSpace
          expr: |
            100 - (windows_logical_disk_free_bytes{volume="C:"} /
            windows_logical_disk_size_bytes{volume="C:"} * 100) > 85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Windows node {{ $labels.instance }} C: drive > 85% full"

        # Windows service down
        - alert: WindowsServiceDown
          expr: windows_service_state{state="running"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Windows service {{ $labels.name }} not running on {{ $labels.instance }}"
```

## Step 5: Key Prometheus Queries for Windows Nodes

```promql
# CPU usage per Windows node
100 - (avg by (instance) (
  rate(windows_cpu_time_total{mode="idle"}[5m])
) * 100)

# Memory usage percentage
(1 - windows_os_physical_memory_free_bytes /
windows_cs_physical_memory_bytes) * 100

# Disk I/O read bytes/sec
rate(windows_logical_disk_read_bytes_total{volume="C:"}[5m])

# Network throughput
rate(windows_net_bytes_received_total[5m]) +
rate(windows_net_bytes_sent_total[5m])

# Container CPU usage on Windows
rate(windows_container_cpu_usage_seconds_total[5m])

# Container memory usage
windows_container_memory_usage_private_working_set_bytes
```

## Step 6: Import Grafana Dashboard

```bash
# Import pre-built Windows Exporter dashboard
# Dashboard ID: 14694 (Windows Node Exporter Full)

# Or create custom dashboard via Grafana API
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -d '{
    "dashboard": {
      "title": "Rancher Windows Nodes",
      "panels": [
        {
          "title": "CPU Usage",
          "type": "graph",
          "targets": [{
            "expr": "100 - (avg by (instance) (rate(windows_cpu_time_total{mode=\"idle\"}[5m])) * 100)"
          }]
        }
      ]
    },
    "overwrite": true
  }' \
  "http://grafana.cattle-monitoring-system.svc:3000/api/dashboards/db"
```

## Step 7: Monitor Windows Container Metrics

```promql
# Windows container CPU (per container)
rate(windows_container_cpu_usage_seconds_total[5m])
* on(container_id) group_left(name, namespace, pod)
kube_pod_container_info{container_id=~"container.*"}

# Container restarts on Windows pods
kube_pod_container_status_restarts_total{
  node=~"win-.*"  # Match Windows node names
}
```

## Conclusion

Monitoring Windows nodes in Rancher requires the Windows Exporter DaemonSet as a bridge between Windows performance counters and Prometheus. The Windows Exporter exposes CPU, memory, disk, network, and container metrics in Prometheus format. Combined with the Grafana dashboard for Windows nodes and targeted alerts for critical thresholds, you gain the same level of observability for Windows nodes as for Linux nodes. Ensure the DaemonSet tolerates Windows node taints so it deploys on every Windows worker.
