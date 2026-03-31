# How to Set Up Prometheus Service Monitors for Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Prometheus, Monitoring, Kubernetes

Description: Configure Prometheus ServiceMonitor resources to scrape Rook-Ceph metrics and enable comprehensive storage cluster observability.

---

## Overview

Rook-Ceph exposes rich metrics through the Ceph Manager's Prometheus module. The Prometheus Operator uses `ServiceMonitor` resources to define which services to scrape. Rook can automatically create these ServiceMonitors, or you can configure them manually for more control.

## Prerequisites

- Prometheus Operator installed (kube-prometheus-stack or standalone)
- Rook-Ceph running with the Ceph MGR Prometheus module enabled
- Prometheus Operator watching the `rook-ceph` namespace

Verify Prometheus Operator CRDs:

```bash
kubectl get crd | grep monitoring.coreos.com
```

## Enable Automatic ServiceMonitor Creation

Rook can automatically create ServiceMonitors when monitoring is enabled in the CephCluster CRD:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  monitoring:
    enabled: true
    interval: 10s
```

Apply the change:

```bash
kubectl apply -f cephcluster.yaml
kubectl get servicemonitor -n rook-ceph
```

## Manual ServiceMonitor Configuration

For clusters where you want explicit control, create the ServiceMonitor manually:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rook-ceph-mgr
  namespace: rook-ceph
  labels:
    team: rook
spec:
  namespaceSelector:
    matchNames:
      - rook-ceph
  selector:
    matchLabels:
      app: rook-ceph-mgr
      rook_cluster: rook-ceph
  endpoints:
  - port: http-metrics
    path: /metrics
    interval: 5s
    scheme: http
```

```bash
kubectl apply -f rook-servicemonitor.yaml
```

## Configure Prometheus to Watch rook-ceph Namespace

If Prometheus is installed in a different namespace, update its ServiceMonitor selector to include the `rook-ceph` namespace:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: kube-prometheus
  namespace: monitoring
spec:
  serviceMonitorNamespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: rook-ceph
  serviceMonitorSelector:
    matchLabels:
      team: rook
```

Alternatively, label the `rook-ceph` namespace:

```bash
kubectl label namespace rook-ceph prometheus=enabled
```

## Verify Metrics are Being Scraped

Check that Prometheus has discovered the Rook targets:

```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &

# Query a Ceph metric
curl -s 'http://localhost:9090/api/v1/query?query=ceph_health_status' | jq .
```

Expected output showing Ceph health metric:

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {"__name__": "ceph_health_status", "instance": "..."},
        "value": [1700000000, "0"]
      }
    ]
  }
}
```

## Summary

Setting up Prometheus ServiceMonitors for Rook-Ceph enables comprehensive storage monitoring through the Prometheus Operator ecosystem. Whether you use the automatic ServiceMonitor creation via the CephCluster CRD or define ServiceMonitors manually, the result is a full set of Ceph metrics - pool usage, OSD performance, IOPS, latency - available for alerting and Grafana dashboards.
