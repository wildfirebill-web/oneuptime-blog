# How to Monitor RKE2 Cluster Health

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Rancher, Monitoring, Observability, Prometheus

Description: Learn how to monitor the health of your RKE2 cluster using built-in tools, Prometheus, Grafana, and OneUptime.

## Introduction

Maintaining cluster health is critical for production Kubernetes deployments. RKE2 exposes a rich set of metrics and status endpoints that you can use to monitor the state of your cluster components. This guide covers multiple monitoring approaches, from basic kubectl commands to full observability stacks.

## Basic Health Checks with kubectl

Start with the built-in Kubernetes health check endpoints.

```bash
# Set up kubectl (if not already done)

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

# Check all node statuses
kubectl get nodes -o wide

# Check the health of system components
kubectl get componentstatuses

# Check all pods across all namespaces
kubectl get pods --all-namespaces

# Check for any pods that are not running
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed
```

## Checking API Server Health

```bash
# Check API server liveness endpoint
curl -sk https://localhost:6443/livez
# Expected: ok

# Check API server readiness
curl -sk https://localhost:6443/readyz
# Expected: ok

# Check individual health components
curl -sk "https://localhost:6443/readyz?verbose"
```

## Checking etcd Health

```bash
# Use etcdctl to check cluster health
sudo /var/lib/rancher/rke2/bin/etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
    --key=/var/lib/rancher/rke2/server/tls/etcd/client.key \
    endpoint health

# Check etcd member list
sudo /var/lib/rancher/rke2/bin/etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
    --key=/var/lib/rancher/rke2/server/tls/etcd/client.key \
    member list

# Check etcd endpoint status (shows leader and DB size)
sudo /var/lib/rancher/rke2/bin/etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
    --key=/var/lib/rancher/rke2/server/tls/etcd/client.key \
    endpoint status --write-out=table
```

## Monitoring Node Resources

```bash
# Check node resource usage (requires metrics-server)
kubectl top nodes

# Check pod resource usage
kubectl top pods --all-namespaces --sort-by=memory

# Describe a node to see capacity, allocations, and events
kubectl describe node <NODE_NAME>
```

## Deploying the RKE2 Monitoring Stack

RKE2 includes a built-in monitoring chart based on kube-prometheus-stack.

```bash
# Install the RKE2 monitoring chart
helm repo add rancher-charts https://charts.rancher.io
helm repo update

# Install kube-prometheus-stack
helm install rancher-monitoring rancher-charts/rancher-monitoring \
    --namespace cattle-monitoring-system \
    --create-namespace \
    --version 103.0.0 \
    --set prometheus.prometheusSpec.retention=15d \
    --set prometheus.prometheusSpec.retentionSize=10GB \
    --set grafana.adminPassword=admin123
```

### Access Grafana

```bash
# Port-forward to Grafana
kubectl -n cattle-monitoring-system port-forward \
    svc/rancher-monitoring-grafana 3000:80

# Or get the LoadBalancer IP
kubectl -n cattle-monitoring-system get svc rancher-monitoring-grafana
```

Access Grafana at `http://localhost:3000` with default credentials `admin/admin123`.

## Deploying Prometheus Manually

If you prefer a standalone Prometheus deployment:

```yaml
# prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 30s
      evaluation_interval: 30s

    scrape_configs:
      # Scrape Kubernetes API server metrics
      - job_name: kubernetes-apiservers
        kubernetes_sd_configs:
          - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name]
            action: keep
            regex: default;kubernetes

      # Scrape node metrics via node-exporter
      - job_name: node-exporter
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
```

## Setting Up Alerts

Create Prometheus alert rules for critical cluster conditions:

```yaml
# cluster-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cluster-health-alerts
  namespace: monitoring
spec:
  groups:
    - name: cluster.health
      rules:
        # Alert when a node is not ready for more than 5 minutes
        - alert: NodeNotReady
          expr: kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Node {{ $labels.node }} is not ready"
            description: "Node {{ $labels.node }} has been NotReady for more than 5 minutes."

        # Alert when etcd has no leader
        - alert: EtcdNoLeader
          expr: etcd_server_has_leader == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "etcd cluster has no leader"

        # Alert when API server is down
        - alert: KubeAPIServerDown
          expr: absent(up{job="kubernetes-apiservers"} == 1)
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Kubernetes API server is down"

        # Alert when node memory is > 90%
        - alert: NodeHighMemory
          expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High memory usage on {{ $labels.instance }}"
```

## Health Check Script

Create a comprehensive health check script for quick status reviews:

```bash
#!/bin/bash
# rke2-health-check.sh

KUBECONFIG=/etc/rancher/rke2/rke2.yaml
KUBECTL="/var/lib/rancher/rke2/bin/kubectl --kubeconfig $KUBECONFIG"
ETCDCTL="/var/lib/rancher/rke2/bin/etcdctl"
ETCD_ARGS="--endpoints=https://127.0.0.1:2379 \
    --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
    --key=/var/lib/rancher/rke2/server/tls/etcd/client.key"

echo "=== Node Status ==="
$KUBECTL get nodes -o wide

echo ""
echo "=== System Pod Status ==="
$KUBECTL get pods -n kube-system

echo ""
echo "=== etcd Health ==="
sudo $ETCDCTL $ETCD_ARGS endpoint health

echo ""
echo "=== API Server Health ==="
curl -sk https://localhost:6443/livez && echo " [OK]" || echo " [FAILED]"

echo ""
echo "=== Failed Pods ==="
$KUBECTL get pods --all-namespaces | grep -v Running | grep -v Completed | grep -v Pending
```

```bash
chmod +x rke2-health-check.sh
sudo ./rke2-health-check.sh
```

## Conclusion

Monitoring RKE2 cluster health requires a layered approach: starting with kubectl for immediate visibility, using etcdctl for data store health, and deploying Prometheus with alerting rules for proactive monitoring. The combination of built-in health endpoints, the RKE2 monitoring stack, and custom alert rules gives you comprehensive visibility into your cluster's state. Set up automated alerts so you can respond to issues before they impact workloads.
