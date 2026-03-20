# How to Configure Harvester Monitoring - Setup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Monitoring, Prometheus, Grafana, Kubernetes, HCI, SUSE Rancher

Description: Learn how to enable and configure the Harvester monitoring stack including Prometheus, Alertmanager, and Grafana dashboards for comprehensive HCI visibility.

---

Harvester includes a built-in monitoring system based on the kube-prometheus-stack. This guide covers enabling it, customizing retention, and accessing Grafana dashboards.

---

## Step 1: Enable Monitoring in Harvester

In the Harvester UI:

1. Go to **Advanced > Monitoring & Logging**
2. Click **Enable Monitoring**
3. Configure storage and retention settings

Or via Helm (if managing Harvester via Rancher):

```bash
helm repo add rancher-charts https://charts.rancher.io
helm install rancher-monitoring \
  rancher-charts/rancher-monitoring \
  --namespace cattle-monitoring-system \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=longhorn \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

---

## Step 2: Configure Prometheus Retention

```yaml
# Patch Prometheus to adjust retention

kubectl patch prometheus rancher-monitoring-prometheus \
  -n cattle-monitoring-system \
  --type merge \
  -p '{
    "spec": {
      "retention": "30d",
      "retentionSize": "45GB"
    }
  }'
```

---

## Step 3: Configure Alertmanager

```yaml
# alertmanager-config.yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: harvester-alerts
  namespace: cattle-monitoring-system
spec:
  route:
    receiver: default
    groupBy: [alertname, cluster]
    routes:
      - receiver: slack-ops
        match:
          severity: warning
      - receiver: pagerduty-critical
        match:
          severity: critical

  receivers:
    - name: default
    - name: slack-ops
      slackConfigs:
        - apiURL:
            name: slack-webhook-secret
            key: url
          channel: '#harvester-alerts'
          title: 'Harvester Alert: {{ .GroupLabels.alertname }}'
    - name: pagerduty-critical
      pagerdutyConfigs:
        - serviceKey:
            name: pagerduty-secret
            key: key
```

---

## Step 4: Access Grafana

```bash
# Get Grafana credentials
kubectl get secret rancher-monitoring-grafana \
  -n cattle-monitoring-system \
  -o jsonpath='{.data.admin-password}' | base64 -d

# Port-forward Grafana
kubectl port-forward svc/rancher-monitoring-grafana \
  -n cattle-monitoring-system 3000:80

# Access at http://localhost:3000
```

Pre-built Harvester dashboards are available under the Grafana **Harvester** folder.

---

## Step 5: Import Harvester-Specific Dashboards

Import these Grafana dashboard IDs for comprehensive Harvester visibility:

| Dashboard | ID | Description |
|---|---|---|
| Harvester Overview | 17119 | Cluster-wide metrics |
| Longhorn Volumes | 17626 | Storage health |
| Node Exporter Full | 1860 | Per-node system metrics |
| KubeVirt VMs | 12006 | VM performance metrics |

```bash
# Import via Grafana API
curl -X POST http://admin:password@localhost:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  -d '{"dashboardId": 17119, "overwrite": true, "inputs": [{"name": "DS_PROMETHEUS", "type": "datasource", "pluginId": "prometheus", "value": "Prometheus"}]}'
```

---

## Best Practices

- Use Longhorn-backed persistent volumes for Prometheus data to survive node failures.
- Set Prometheus retention based on your compliance requirements - typically 30-90 days.
- Configure separate Alertmanager routes for node-level alerts vs. VM-level alerts.
- Add custom dashboards for your specific VM workloads alongside the default Harvester dashboards.
