# How to View Longhorn Dashboard in Grafana

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, Grafana, Monitoring, Visualization

Description: Set up Grafana dashboards for Longhorn storage monitoring, including how to import the official Longhorn dashboard and create custom panels for capacity and health metrics.

## Introduction

Grafana provides powerful visualization capabilities for Longhorn metrics collected by Prometheus. The Longhorn community provides an official Grafana dashboard that displays volume health, disk usage, backup status, and performance metrics in an easy-to-understand format. This guide covers importing the official dashboard and customizing it for your environment.

## Prerequisites

- Prometheus configured to scrape Longhorn metrics (see the Longhorn Prometheus guide)
- Grafana installed and connected to the Prometheus data source
- Longhorn metrics visible in Prometheus

## Installing Grafana (if not already installed)

```bash
# Add the Grafana Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Grafana
helm install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  --set persistence.enabled=true \
  --set persistence.storageClassName=longhorn \
  --set persistence.size=10Gi

# Get the admin password
kubectl get secret --namespace monitoring grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode

# Access Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80
```

## Connecting Grafana to Prometheus

1. Open Grafana at `http://localhost:3000`
2. Login with `admin` and the password from above
3. Navigate to **Configuration** → **Data Sources**
4. Click **Add data source**
5. Select **Prometheus**
6. Enter the Prometheus URL: `http://prometheus.monitoring.svc:9090`
7. Click **Save & Test**

## Importing the Official Longhorn Dashboard

### Method 1: Import by Dashboard ID

1. In Grafana, click the **+** icon → **Import**
2. Enter Dashboard ID `16888` (official Longhorn dashboard)
3. Click **Load**
4. Select your Prometheus data source
5. Click **Import**

### Method 2: Import from JSON

Download the official dashboard JSON:

```bash
# Download the official Longhorn Grafana dashboard
curl -sSfL \
  "https://grafana.com/api/dashboards/16888/revisions/latest/download" \
  -o longhorn-dashboard.json
```

1. In Grafana, click **+** → **Import**
2. Click **Upload JSON file**
3. Select the downloaded `longhorn-dashboard.json`
4. Configure the data source
5. Click **Import**

### Method 3: Via Grafana ConfigMap (GitOps)

```yaml
# grafana-longhorn-dashboard.yaml - Dashboard as a ConfigMap for Grafana sidecar
apiVersion: v1
kind: ConfigMap
metadata:
  name: longhorn-dashboard
  namespace: monitoring
  labels:
    # This label causes Grafana's dashboard provisioner to automatically load it
    grafana_dashboard: "1"
data:
  longhorn-dashboard.json: |
    {
      "__requires": [{"type": "grafana", "id": "grafana", "version": "8.0.0"}],
      "annotations": {},
      "description": "Longhorn Storage Dashboard",
      "panels": []
    }
```

```bash
kubectl apply -f grafana-longhorn-dashboard.yaml
```

## Creating Custom Dashboard Panels

### Panel 1: Volume Health Status

```json
{
  "title": "Volume Health",
  "type": "stat",
  "targets": [
    {
      "expr": "count(longhorn_volume_robustness == 1)",
      "legendFormat": "Healthy"
    },
    {
      "expr": "count(longhorn_volume_robustness == 2)",
      "legendFormat": "Degraded"
    },
    {
      "expr": "count(longhorn_volume_robustness == 3)",
      "legendFormat": "Faulted"
    }
  ]
}
```

In Grafana:
1. Click **Add panel** on your dashboard
2. Select **Stat** visualization
3. Add the PromQL expression: `count(longhorn_volume_robustness == 1)`

### Panel 2: Disk Usage Percentage

```promql
# PromQL for disk usage percentage per node
(1 - (longhorn_disk_storage_available_bytes / longhorn_disk_storage_maximum_bytes)) * 100
```

Configuration:
- Visualization: **Gauge**
- Min: 0, Max: 100
- Thresholds: Green (0-70), Yellow (70-90), Red (90-100)

### Panel 3: Volume I/O Throughput

```promql
# Read throughput (bytes per second)
rate(longhorn_volume_read_throughput[5m])

# Write throughput (bytes per second)
rate(longhorn_volume_write_throughput[5m])
```

Configuration:
- Visualization: **Time series**
- Unit: `bytes/sec`
- Legend: Volume name

### Panel 4: Backup Status

```promql
# Number of volumes with recent backups
longhorn_manager_backup_volume_count

# Total backup count
longhorn_manager_backup_count
```

## Setting Up Dashboard Alerts in Grafana

Grafana can also send alerts based on dashboard metrics:

```yaml
# Grafana alert rule for degraded volumes
alertName: "Longhorn Volume Degraded"
condition: "Last() of (count by() (longhorn_volume_robustness == 2)) > 0"
evaluation:
  interval: 1m
  pending: 5m
notifications:
  - channel: slack-alerts
  - channel: pagerduty
```

In Grafana UI:
1. Open your dashboard panel
2. Click the panel title → **Edit**
3. Go to the **Alert** tab
4. Configure conditions, evaluation period, and notification channels

## Dashboard Variables for Multi-Cluster

If you monitor multiple clusters, add a cluster variable:

1. Go to Dashboard **Settings** → **Variables**
2. Add a new variable:
   - **Name**: `cluster`
   - **Type**: Query
   - **Query**: `label_values(longhorn_volume_state, cluster)`
3. Use `$cluster` in your PromQL queries: `longhorn_volume_state{cluster="$cluster"}`

## Exporting Your Custom Dashboard

After creating custom panels, export your dashboard for GitOps or sharing:

1. Click the **Share** icon → **Export**
2. Click **Save to file**
3. Store the JSON in your GitOps repository

```bash
# Store dashboard as a ConfigMap for automated provisioning
kubectl create configmap longhorn-custom-dashboard \
  --from-file=longhorn-custom.json \
  -n monitoring \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Conclusion

Grafana dashboards transform raw Longhorn Prometheus metrics into actionable visual insights. The official Longhorn dashboard provides a solid starting point, while custom panels allow you to focus on the metrics most relevant to your environment. Combined with Prometheus alerting rules, Grafana dashboards give your operations team the visibility needed to maintain healthy, well-managed Kubernetes storage infrastructure.
