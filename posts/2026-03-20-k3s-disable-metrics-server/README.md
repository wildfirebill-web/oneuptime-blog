# How to Disable Metrics Server in K3s

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Kubernetes, Rancher, Metrics Server, Prometheus, Monitoring

Description: Learn how to disable K3s's built-in metrics-server and replace it with a custom Prometheus-based metrics stack.

## Introduction

K3s includes the Kubernetes Metrics Server by default, which provides the `kubectl top` commands and powers Horizontal Pod Autoscaler (HPA). Some teams prefer to disable the built-in metrics-server and replace it with a more comprehensive metrics solution like Prometheus with the Prometheus Adapter, which can serve both HPA metrics and custom application metrics.

## Understanding the Metrics Server

The default K3s metrics-server:
- Collects CPU and memory metrics from kubelets
- Powers `kubectl top nodes` and `kubectl top pods`
- Is required by the Horizontal Pod Autoscaler (HPA) with default metrics
- Is lightweight but limited to basic resource metrics

## Reasons to Replace the Metrics Server

1. You want custom HPA metrics (e.g., scale on request rate, queue depth)
2. You're deploying a full Prometheus stack and want a unified metrics source
3. You need historical metrics storage (Prometheus + Thanos/Cortex)
4. Resource constraint - the metrics server consumes ~15MB RAM

## Disabling the Metrics Server

### Before Installation

```bash
sudo mkdir -p /etc/rancher/k3s

sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
token: "ClusterToken"
disable:
  - metrics-server
EOF

curl -sfL https://get.k3s.io | sudo sh -
```

### On an Existing Cluster

```bash
# Add metrics-server to the disable list

echo "  - metrics-server" >> /etc/rancher/k3s/config.yaml

sudo systemctl restart k3s

# Verify removal
kubectl -n kube-system get pods | grep metrics-server
# Should return empty
```

## Verifying Metrics Server is Removed

```bash
# Check no metrics-server pods are running
kubectl -n kube-system get pods | grep metrics-server

# Verify kubectl top no longer works (expected)
kubectl top nodes
# Error: metrics not available yet

# Check APIService is gone
kubectl get apiservice | grep metrics
# v1beta1.metrics.k8s.io should be gone
```

## Option 1: Install Metrics Server Manually

If you just want to customize the metrics-server (not replace it):

```bash
# Install a specific version with custom settings
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update

helm install metrics-server metrics-server/metrics-server \
    --namespace kube-system \
    --set args[0]=--kubelet-preferred-address-types=InternalIP \
    --set args[1]=--kubelet-insecure-tls  # Only for development
```

### For K3s with Self-Signed Certs (Production)

```bash
helm install metrics-server metrics-server/metrics-server \
    --namespace kube-system \
    --set args[0]=--kubelet-preferred-address-types=InternalIP \
    --set args[1]=--kubelet-certificate-authority=/var/lib/rancher/k3s/agent/server-ca.crt
```

## Option 2: Install Full Prometheus Stack

For production monitoring, deploy kube-prometheus-stack:

```bash
# Add the Prometheus community Helm charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --create-namespace \
    --set grafana.adminPassword=admin123 \
    --set prometheus.prometheusSpec.retention=30d \
    --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=20Gi
```

## Option 3: Install Prometheus Adapter for HPA

If you disabled metrics-server but need HPA to work with Prometheus metrics:

```bash
# Install the Prometheus Adapter
helm install prometheus-adapter prometheus-community/prometheus-adapter \
    --namespace monitoring \
    --set prometheus.url=http://prometheus-operated.monitoring.svc.cluster.local \
    --set prometheus.port=9090

# Verify the API is registered
kubectl get apiservice | grep custom.metrics
kubectl get apiservice | grep external.metrics
```

### Configure Custom HPA Metrics

```yaml
# hpa-custom-metrics.yaml
# First define the metrics in prometheus-adapter config
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
    - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "^(.*)_total"
        as: "${1}_per_second"
      metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

```yaml
# hpa-with-custom-metric.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
    # Standard CPU metric (from metrics-server or prometheus-adapter)
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
    # Custom metric from Prometheus
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
```

## Restoring kubectl top Functionality

After deploying either the standalone metrics-server or kube-prometheus-stack:

```bash
# Test kubectl top with standalone metrics-server
kubectl top nodes
kubectl top pods --all-namespaces

# Or with prometheus-adapter
kubectl top nodes  # Works once prometheus-adapter serves resource metrics
```

## Comparing Solutions

| Feature | K3s Metrics Server | Standalone Metrics Server | kube-prometheus-stack |
|---------|--------------------|--------------------------|----------------------|
| kubectl top | Yes | Yes | Yes (with adapter) |
| HPA (CPU/Memory) | Yes | Yes | Yes |
| HPA (Custom) | No | No | Yes |
| Long-term storage | No | No | Yes |
| Dashboards | No | No | Grafana |
| Alerting | No | No | Alertmanager |
| Resource usage | Low | Low | Medium-High |

## Conclusion

Disabling K3s's built-in metrics-server makes sense when you're deploying a comprehensive Prometheus-based monitoring stack and want a unified metrics source. For basic `kubectl top` and HPA functionality without Prometheus, reinstalling the standalone metrics-server with Helm gives you full control over the configuration. For production clusters requiring custom HPA metrics, the Prometheus Adapter bridges Prometheus metrics to the Kubernetes metrics API, enabling sophisticated autoscaling based on any metric Prometheus can collect.
