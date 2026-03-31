# How to Import Grafana Dashboards for Rook-Ceph (IDs: 2842, 5336, 5342)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Grafana, Dashboard, Monitoring

Description: Import and configure the official Ceph Grafana dashboards (IDs 2842, 5336, 5342) for Rook-managed clusters using ConfigMaps or the Grafana UI.

---

## Overview

Grafana.com hosts several community-maintained Ceph dashboards that provide rich visualizations for Rook-managed clusters. The most commonly used are:

- **2842** - Ceph - Cluster (overall cluster health and capacity)
- **5336** - Ceph - OSD (per-OSD performance and utilization)
- **5342** - Ceph - Pools (per-pool IOPS and throughput)

## Option 1 - Import via Grafana UI

1. Open Grafana and navigate to `Dashboards` - `Import`
2. Enter the dashboard ID (e.g., `2842`) and click `Load`
3. Select your Prometheus data source from the dropdown
4. Click `Import`

Repeat for IDs 5336 and 5342.

## Option 2 - Import via ConfigMap (Grafana Sidecar)

If Grafana is deployed with the sidecar dashboard loader (as in kube-prometheus-stack), create ConfigMaps with the dashboard JSON:

```bash
# Download the dashboard JSON
curl -o ceph-cluster.json \
  "https://grafana.com/api/dashboards/2842/revisions/latest/download"

# Create ConfigMap for Grafana sidecar
kubectl create configmap grafana-dashboard-ceph-cluster \
  --from-file=ceph-cluster.json \
  -n monitoring
kubectl label configmap grafana-dashboard-ceph-cluster \
  grafana_dashboard=1 -n monitoring
```

Repeat for the other dashboards:

```bash
for id in 5336 5342; do
  curl -o ceph-${id}.json "https://grafana.com/api/dashboards/${id}/revisions/latest/download"
  kubectl create configmap grafana-dashboard-ceph-${id} \
    --from-file=ceph-${id}.json -n monitoring
  kubectl label configmap grafana-dashboard-ceph-${id} grafana_dashboard=1 -n monitoring
done
```

## Option 3 - kube-prometheus-stack Helm Values

If using the kube-prometheus-stack Helm chart, configure dashboard downloads in values:

```yaml
grafana:
  dashboards:
    default:
      ceph-cluster:
        gnetId: 2842
        revision: 17
        datasource: Prometheus
      ceph-osd:
        gnetId: 5336
        revision: 9
        datasource: Prometheus
      ceph-pools:
        gnetId: 5342
        revision: 9
        datasource: Prometheus
```

Upgrade the chart:

```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring -f values.yaml
```

## Configure the Prometheus Data Source

Ensure Grafana is pointed at the correct Prometheus instance. In Grafana, go to `Configuration` - `Data Sources` and verify the Prometheus URL:

```text
http://prometheus-operated.monitoring.svc.cluster.local:9090
```

## Verify Dashboard Data

After import, navigate to each dashboard and confirm panels are populating:

```bash
# Check that Ceph metrics exist in Prometheus
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090 &
curl -s 'http://localhost:9090/api/v1/label/__name__/values' | jq '.data[] | select(startswith("ceph_"))'
```

## Summary

The Grafana dashboards with IDs 2842, 5336, and 5342 provide comprehensive visualization for Rook-Ceph clusters. Whether imported via the UI, ConfigMap sidecar injection, or the kube-prometheus-stack Helm chart, these dashboards surface cluster health, OSD performance, and pool utilization from the Prometheus metrics Rook exposes.
