# How to Configure Harvester Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Monitoring, Prometheus, Grafana

Description: Learn how to set up and configure Harvester's built-in monitoring stack with Prometheus and Grafana for comprehensive cluster observability.

## Introduction

Harvester includes a built-in monitoring stack based on Prometheus and Grafana. This monitoring solution collects metrics from all cluster components — nodes, VMs, storage, and networking — and provides pre-built dashboards for visualization. This guide covers enabling monitoring, accessing Grafana dashboards, configuring alerting rules, and integrating with external monitoring systems.

## Harvester Monitoring Components

- **Prometheus**: Time-series metrics collection and storage
- **Grafana**: Visualization and dashboarding
- **Alertmanager**: Alert routing and notification
- **node-exporter**: Node-level OS metrics
- **KubeVirt metrics**: VM-specific metrics
- **Longhorn metrics**: Storage metrics

## Step 1: Enable Monitoring via the UI

1. Navigate to **Advanced** → **Monitoring & Logging**
2. Click **Enable Monitoring**
3. Configure:
   - **Prometheus Storage**: Set retention and storage size
   - **Grafana**: Enable/disable public access
4. Click **Save**

## Step 2: Enable Monitoring via kubectl

```yaml
# harvester-monitoring.yaml
# Enable and configure Harvester monitoring

apiVersion: harvesterhci.io/v1beta1
kind: Setting
metadata:
  name: harvester-monitoring
  namespace: harvester-system
spec:
  value: |
    {
      "enabled": true,
      "prometheusNodeExporter": {
        "resources": {
          "requests": {
            "cpu": "150m",
            "memory": "100Mi"
          }
        }
      },
      "prometheus": {
        "prometheusSpec": {
          "evaluationInterval": "1m",
          "scrapeInterval": "1m",
          "retention": "5d",
          "retentionSize": "50GiB",
          "resources": {
            "requests": {
              "cpu": "750m",
              "memory": "750Mi"
            }
          }
        }
      },
      "grafana": {
        "enabled": true
      }
    }
```

```bash
# Or install via Helm (Harvester's monitoring chart)
helm repo add rancher-charts https://charts.rancher.io
helm repo update

helm install rancher-monitoring rancher-charts/rancher-monitoring \
    --namespace cattle-monitoring-system \
    --create-namespace \
    --set prometheus.prometheusSpec.retentionSize="50GiB" \
    --set prometheus.prometheusSpec.retention="5d" \
    --set grafana.enabled=true \
    --set alertmanager.enabled=true
```

## Step 3: Access the Grafana Dashboard

```bash
# Get the Grafana service URL
kubectl get svc -n cattle-monitoring-system | grep grafana

# Forward the Grafana port to your local machine
kubectl port-forward -n cattle-monitoring-system \
    svc/rancher-monitoring-grafana 3000:80

# Access Grafana at http://localhost:3000
# Default credentials: admin / (get from secret)
kubectl get secret -n cattle-monitoring-system rancher-monitoring-grafana \
    -o jsonpath='{.data.admin-password}' | base64 -d
```

## Step 4: Key Grafana Dashboards for Harvester

Navigate to these pre-built dashboards in Grafana:

### Node Dashboards
- **Kubernetes / Nodes**: CPU, memory, disk, network per node
- **Node Exporter Full**: Detailed OS-level metrics

### VM Dashboards

```bash
# Import the KubeVirt dashboard
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubevirt-grafana-dashboard
  namespace: cattle-monitoring-system
  labels:
    grafana_dashboard: "1"
data:
  kubevirt-dashboard.json: |
    {
      "title": "KubeVirt VM Metrics",
      "uid": "kubevirt-vms",
      "panels": [
        {
          "title": "VM CPU Usage",
          "type": "graph",
          "datasource": "Prometheus",
          "targets": [
            {
              "expr": "sum by (name, namespace) (rate(kubevirt_vmi_cpu_usage_seconds_total[5m]))",
              "legendFormat": "{{namespace}}/{{name}}"
            }
          ]
        }
      ]
    }
EOF
```

### Storage Dashboards
- **Longhorn**: Volume health, replica status, disk utilization

## Step 5: Configure Prometheus Alerting Rules

Create custom alert rules for Harvester-specific conditions:

```yaml
# harvester-alert-rules.yaml
# Custom Prometheus alerting rules for Harvester

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: harvester-alerts
  namespace: cattle-monitoring-system
  labels:
    app: kube-prometheus-stack
    release: rancher-monitoring
spec:
  groups:
    - name: harvester.node
      interval: 30s
      rules:
        # Alert when a node is not ready
        - alert: HarvesterNodeNotReady
          expr: kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Harvester node {{ $labels.node }} is not ready"
            description: "Node {{ $labels.node }} has been not ready for 5 minutes"

        # Alert when node memory is critical
        - alert: HarvesterNodeMemoryCritical
          expr: |
            (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 0.95
          for: 10m
          labels:
            severity: critical
          annotations:
            summary: "Node {{ $labels.instance }} memory is critically full"
            description: "Node memory usage exceeds 95%"

        # Alert when node disk is nearly full
        - alert: HarvesterNodeDiskFull
          expr: |
            (1 - (node_filesystem_free_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) > 0.85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Node {{ $labels.instance }} root disk is nearly full"

    - name: harvester.storage
      rules:
        # Alert when a Longhorn volume is degraded
        - alert: LonghornVolumeDegraded
          expr: longhorn_volume_robustness == 2
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Longhorn volume {{ $labels.volume }} is degraded"

        # Alert when a Longhorn volume is faulted
        - alert: LonghornVolumeFaulted
          expr: longhorn_volume_robustness == 3
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Longhorn volume {{ $labels.volume }} is FAULTED - data may be at risk"

        # Alert when storage utilization > 80%
        - alert: LonghornStorageAlmostFull
          expr: |
            (longhorn_node_storage_usage_bytes / longhorn_node_storage_capacity_bytes) > 0.80
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Longhorn storage on node {{ $labels.node }} is > 80% full"

    - name: harvester.vms
      rules:
        # Alert on VM in errored state
        - alert: HarvesterVMFailed
          expr: kubevirt_vmi_info{phase="Failed"} > 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "VM {{ $labels.name }} in namespace {{ $labels.namespace }} has failed"
```

```bash
kubectl apply -f harvester-alert-rules.yaml

# Verify the rules are loaded
kubectl get prometheusrule -n cattle-monitoring-system
kubectl get prometheusrule harvester-alerts -n cattle-monitoring-system -o yaml | \
    grep -A 5 "status:"
```

## Step 6: Configure Alertmanager for Notifications

```yaml
# alertmanager-config.yaml
# Configure Alertmanager to send alerts to Slack and PagerDuty

apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-rancher-monitoring-alertmanager
  namespace: cattle-monitoring-system
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
      # Slack webhook URL
      slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'default'
      routes:
        # Critical alerts go to PagerDuty
        - match:
            severity: critical
          receiver: pagerduty
        # Warnings go to Slack
        - match:
            severity: warning
          receiver: slack

    receivers:
      - name: 'default'
        slack_configs:
          - channel: '#harvester-alerts'
            title: 'Harvester Alert'
            text: '{{ .CommonAnnotations.summary }}'

      - name: 'slack'
        slack_configs:
          - channel: '#harvester-alerts'
            send_resolved: true
            title: '{{ .CommonLabels.alertname }}'
            text: '{{ .CommonAnnotations.description }}'

      - name: 'pagerduty'
        pagerduty_configs:
          - routing_key: 'YOUR_PAGERDUTY_ROUTING_KEY'
            description: '{{ .CommonAnnotations.summary }}'
```

```bash
kubectl apply -f alertmanager-config.yaml
```

## Step 7: Key Prometheus Queries for Harvester

```promql
# Total VMs by state
count by (phase) (kubevirt_vmi_info)

# VM CPU usage per VM
rate(kubevirt_vmi_cpu_usage_seconds_total[5m]) * 100

# VM memory usage
kubevirt_vmi_memory_used_bytes / kubevirt_vmi_memory_available_bytes * 100

# Longhorn volume IOPS
rate(longhorn_volume_read_iops[5m])
rate(longhorn_volume_write_iops[5m])

# Node disk I/O utilization
rate(node_disk_io_time_seconds_total[5m]) * 100

# Cluster-wide storage usage
sum(longhorn_node_storage_usage_bytes) / sum(longhorn_node_storage_capacity_bytes) * 100
```

## Conclusion

Harvester's built-in monitoring stack provides comprehensive visibility into your entire HCI infrastructure. By configuring custom alert rules for VM failures, storage degradation, and resource exhaustion, you ensure your operations team is notified before problems impact users. The integration between Prometheus, Grafana, and Alertmanager creates a complete observability solution. For organizations using existing monitoring infrastructure (Datadog, New Relic, etc.), Prometheus federation allows Harvester metrics to be scraped and included in your centralized monitoring platform.
