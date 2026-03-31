# How to Send Dapr Metrics to Azure Monitor

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Azure Monitor, Metric, Observability, Kubernetes

Description: Configure Dapr to forward metrics to Azure Monitor using the Azure Monitor managed service for Prometheus and Container Insights.

---

Azure Monitor provides managed Prometheus ingestion and Grafana integration for AKS clusters. This guide shows how to send Dapr sidecar and control plane metrics to Azure Monitor without running your own Prometheus instance.

## Architecture

The Azure Monitor managed service for Prometheus scrapes metrics from your cluster and stores them in an Azure Monitor workspace. You can then query them with PromQL via Azure Managed Grafana.

## Step 1 - Enable Managed Prometheus on AKS

```bash
# Enable the monitoring addon on your AKS cluster
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id /subscriptions/<sub-id>/resourceGroups/myRG/providers/microsoft.monitor/accounts/myWorkspace \
  --grafana-resource-id /subscriptions/<sub-id>/resourceGroups/myRG/providers/microsoft.dashboard/grafana/myGrafana
```

This deploys a metrics addon that automatically scrapes pods with the `prometheus.io/scrape: "true"` annotation.

## Step 2 - Annotate Dapr Pods

Ensure your application pods have the correct annotations:

```yaml
metadata:
  annotations:
    dapr.io/enabled: "true"
    dapr.io/app-id: "inventory-service"
    dapr.io/metrics-port: "9090"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
```

## Step 3 - Create a Custom Scrape Config

For control plane components, create a ConfigMap for the metrics addon:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ama-metrics-prometheus-config
  namespace: kube-system
data:
  prometheus-config: |
    global:
      scrape_interval: 30s
    scrape_configs:
      - job_name: dapr-system
        static_configs:
          - targets:
            - dapr-operator.dapr-system:9090
            - dapr-sentry.dapr-system:9090
            - dapr-placement-server.dapr-system:9090
```

```bash
kubectl apply -f ama-metrics-config.yaml
```

## Step 4 - Query Metrics in Azure Monitor

Open the Azure portal and navigate to your Azure Monitor workspace. Use the Metrics Explorer or run PromQL queries:

```
# Request rate via Azure Managed Prometheus
rate(dapr_http_server_request_count[5m])
```

Or use the Azure CLI:

```bash
az monitor metrics list \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/microsoft.monitor/accounts/myWorkspace \
  --metric dapr_http_server_request_count \
  --output table
```

## Step 5 - Set Up Alerts

Create an alert rule for high error rates:

```bash
az monitor scheduled-query create \
  --resource-group myResourceGroup \
  --name "DaprHighErrorRate" \
  --scopes /subscriptions/<sub>/resourceGroups/myRG/providers/microsoft.monitor/accounts/myWorkspace \
  --condition "count > 10" \
  --condition-query "rate(dapr_http_server_request_count{status_code!~'2..'}[5m])" \
  --evaluation-frequency 5m \
  --window-size 10m \
  --severity 2
```

## Step 6 - Import Dapr Dashboard in Azure Managed Grafana

In Azure Managed Grafana, import the official Dapr dashboard:

1. Navigate to your Grafana instance in the Azure portal.
2. Go to Dashboards > Import.
3. Enter Grafana dashboard ID `15401` (Dapr system services) or upload the JSON.

## Summary

Azure Monitor managed Prometheus makes it easy to collect Dapr metrics on AKS without managing your own Prometheus infrastructure. Annotate your pods, configure custom scrape targets for the Dapr control plane, and use Azure Managed Grafana for visualization. This approach reduces operational overhead while providing full PromQL query capabilities.
