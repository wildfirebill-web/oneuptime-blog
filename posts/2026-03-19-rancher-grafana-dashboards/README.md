# How to Configure Grafana Dashboards in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Monitoring, Grafana

Description: Learn how to access, configure, and customize Grafana dashboards in Rancher for cluster and workload monitoring.

Rancher's monitoring stack includes Grafana with pre-built dashboards for Kubernetes cluster monitoring. This guide shows you how to access Grafana, navigate the default dashboards, configure data sources, and customize the Grafana deployment settings.

## Prerequisites

- Rancher v2.6 or later with the Monitoring chart installed.
- Cluster admin or monitoring view permissions.
- A running Prometheus instance in the cluster.

## Step 1: Access Grafana from the Rancher UI

1. Log in to Rancher and select your cluster.
2. In the left navigation menu, click **Monitoring**.
3. Click the **Grafana** link to open the Grafana UI.

Rancher proxies the Grafana UI through its own authentication, so you are automatically logged in with your Rancher credentials.

## Step 2: Explore Default Dashboards

Rancher installs several pre-configured dashboards. Navigate to **Dashboards > Browse** in Grafana to see them. Key dashboards include:

- **Kubernetes / Compute Resources / Cluster** - Overall cluster CPU, memory, and network usage.
- **Kubernetes / Compute Resources / Namespace (Pods)** - Per-namespace resource consumption.
- **Kubernetes / Compute Resources / Node (Pods)** - Per-node resource breakdown.
- **Kubernetes / Compute Resources / Pod** - Individual pod resource usage.
- **Kubernetes / Networking / Cluster** - Network traffic across the cluster.
- **Node Exporter / Nodes** - Detailed node-level OS metrics.
- **etcd** - etcd cluster health and performance metrics.

## Step 3: Configure Grafana Data Sources

By default, Grafana is configured to use the in-cluster Prometheus as its primary data source. To verify or modify data source settings:

1. In Grafana, go to **Configuration > Data Sources**.
2. Click on the **Prometheus** data source.
3. Verify the URL is set to the internal Prometheus service, typically: `http://rancher-monitoring-prometheus.cattle-monitoring-system.svc:9090`.

To add an external data source, click **Add data source** and configure the connection details.

## Step 4: Configure Grafana Resource Limits

Adjust Grafana resource allocation through the monitoring chart values:

```yaml
grafana:
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

Apply the changes by upgrading the rancher-monitoring Helm release.

## Step 5: Enable Persistent Storage for Grafana

By default, Grafana dashboards created through the UI are lost when the pod restarts. To enable persistent storage:

```yaml
grafana:
  persistence:
    enabled: true
    size: 10Gi
    storageClassName: "gp3"
    accessModes:
      - ReadWriteOnce
```

However, the recommended approach in Rancher is to use ConfigMap-based dashboard provisioning, which survives pod restarts without requiring persistent storage.

## Step 6: Configure Grafana via the Helm Chart

Several Grafana settings can be configured through the monitoring chart values:

```yaml
grafana:
  grafana.ini:
    server:
      root_url: "https://rancher.example.com/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-grafana:80/proxy/"
    auth:
      disable_login_form: true
    auth.proxy:
      enabled: true
      header_name: X-WEBAUTH-USER
      header_property: username
      auto_sign_up: true
    users:
      auto_assign_org_role: Viewer
    dashboards:
      default_home_dashboard_path: /tmp/dashboards/rancher-default-home.json
```

## Step 7: Configure Dashboard Refresh Intervals

Set default refresh intervals and time ranges for dashboards:

```yaml
grafana:
  grafana.ini:
    dashboards:
      min_refresh_interval: "10s"
```

Individual dashboards can override this setting. When viewing a dashboard in Grafana, use the time picker in the upper right corner to adjust the time range and the refresh dropdown to set auto-refresh intervals.

## Step 8: Configure Grafana Sidecar for Dashboard Auto-Discovery

Rancher's Grafana deployment uses a sidecar container that watches for ConfigMaps with a specific label and loads them as dashboards. The default label is:

```yaml
grafana:
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      labelValue: "1"
      searchNamespace: cattle-monitoring-system
```

To allow dashboard ConfigMaps from all namespaces:

```yaml
grafana:
  sidecar:
    dashboards:
      searchNamespace: ALL
```

## Step 9: Organize Dashboards into Folders

You can organize dashboards into folders using annotations on ConfigMaps:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-custom-dashboard
  namespace: cattle-monitoring-system
  labels:
    grafana_dashboard: "1"
  annotations:
    grafana_folder: "Custom Dashboards"
data:
  my-dashboard.json: |
    {
      "dashboard": { ... }
    }
```

The `grafana_folder` annotation tells the sidecar which folder to place the dashboard in.

## Step 10: Configure Notification Channels in Grafana

While Rancher's Alertmanager handles most alerting, Grafana can also send notifications directly:

1. In Grafana, go to **Alerting > Contact Points**.
2. Click **Add contact point**.
3. Configure the notification channel (email, Slack, PagerDuty, etc.).
4. Test the notification by clicking **Test**.

Note that Rancher recommends using Alertmanager for alerting rather than Grafana alerts, as Alertmanager integrates more tightly with the Prometheus ecosystem.

## Step 11: Configure Anonymous Access (Optional)

If you want to share dashboard links with users who do not have Rancher accounts:

```yaml
grafana:
  grafana.ini:
    auth.anonymous:
      enabled: true
      org_name: Main Org.
      org_role: Viewer
```

Use this setting with caution in production environments, as it allows unauthenticated access to monitoring data.

## Verifying Grafana Configuration

Check the Grafana pod status and logs:

```bash
kubectl get pods -n cattle-monitoring-system -l app.kubernetes.io/name=grafana

kubectl logs -n cattle-monitoring-system -l app.kubernetes.io/name=grafana -c grafana
```

To verify the sidecar is loading dashboards:

```bash
kubectl logs -n cattle-monitoring-system -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard
```

## Summary

Grafana in Rancher provides a powerful visualization layer for your Kubernetes metrics. The default dashboards cover most common monitoring scenarios, and you can extend them with custom dashboards provisioned through ConfigMaps. Configure resource limits, persistence, and data sources through the monitoring Helm chart values to match your operational requirements.
