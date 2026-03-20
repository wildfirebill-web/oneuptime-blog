# How to View Kubernetes Cluster Details in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Monitoring, Cluster Management, DevOps

Description: Learn how to view and interpret Kubernetes cluster details, resource summaries, and health information in Portainer.

## Introduction

Portainer's Kubernetes environment dashboard gives you a comprehensive view of your cluster's health, resources, and deployed workloads. From node counts to namespace summaries, this centralized view is invaluable for cluster operators. This guide covers navigating the cluster detail views in Portainer.

## Prerequisites

- Portainer with a Kubernetes environment connected
- Admin or operator access

## Step 1: Access the Kubernetes Dashboard

1. Log in to Portainer
2. Click on your Kubernetes environment from the **Home** screen
3. The cluster dashboard loads

## Step 2: Understand the Dashboard Summary

The dashboard displays key metrics:

```
Cluster Overview
──────────────────────────────────────
Nodes:            5 (4 Ready, 1 Unschedulable)
Namespaces:       12
Applications:     47
Services:         38
Volumes:          24
ConfigMaps:       56
Secrets:          31
```

## Step 3: View Cluster Nodes

Click **Cluster → Nodes** to see detailed node information:

```
NAME        STATUS   ROLES          CPU (Req/Limit)    MEMORY (Req/Limit)
master-01   Ready    control-plane  2.1/4.0 cores      4.2/8.0 GiB
worker-01   Ready    worker         3.2/4.0 cores      5.1/8.0 GiB
worker-02   Ready    worker         2.8/4.0 cores      4.8/8.0 GiB
worker-03   Ready    worker         1.9/4.0 cores      3.7/8.0 GiB
worker-04   Ready    worker         2.2/4.0 cores      4.3/8.0 GiB
```

Click on a node to see:
- Node conditions (Ready, MemoryPressure, DiskPressure, PIDPressure)
- Allocated resources
- Labels and taints
- Pods running on the node

## Step 4: View Cluster Resource Usage

```bash
# View resource usage from kubectl
kubectl top nodes

# NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# worker-01     850m         21%    5120Mi          63%
# worker-02     720m         18%    4896Mi          60%
# worker-03     480m         12%    3840Mi          47%
```

In Portainer, the Cluster view shows aggregated resource requests/limits.

## Step 5: View Kubernetes Version Information

In Portainer cluster details:

```
Kubernetes Version:   1.28.4
Platform:             linux/amd64
Container Runtime:    containerd://1.7.3
```

```bash
# CLI equivalent
kubectl version --short
kubectl get nodes -o jsonpath='{.items[0].status.nodeInfo.kubeletVersion}'
```

## Step 6: Check Namespace Overview

Navigate to **Namespaces** for a summary of all namespaces:

```
NAMESPACE           STATUS   WORKLOADS   CPU REQUEST   MEMORY REQUEST
default             Active   3           150m          256Mi
production          Active   18          4500m         8192Mi
staging             Active   12          2000m         4096Mi
monitoring          Active   8           1500m         3072Mi
kube-system         Active   12          800m          1536Mi
```

## Step 7: View Events for Cluster Health

```bash
# View all recent events
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20

# View only warnings
kubectl get events --all-namespaces --field-selector type=Warning
```

In Portainer, navigate to specific namespaces to see events for workloads in that namespace.

## Step 8: Check Cluster Certificates

```bash
# Check certificate expiry (kubeadm clusters)
kubeadm certs check-expiration

# CERTIFICATE                EXPIRES                  RESIDUAL TIME
# admin.conf                 Dec 15, 2024 10:00 UTC   364d
# apiserver                  Dec 15, 2024 10:00 UTC   364d
# apiserver-etcd-client      Dec 15, 2024 10:00 UTC   364d
# apiserver-kubelet-client   Dec 15, 2024 10:00 UTC   364d
```

## Step 9: Inspect Cluster Configuration

```bash
# View cluster-level API resources
kubectl api-resources

# Check storage classes
kubectl get storageclasses

# View persistent volumes
kubectl get pv
```

In Portainer: navigate to **Cluster → Storage Classes** or **Volumes** to see this information visually.

## Step 10: Monitor Control Plane Health

```bash
# Check control plane component health
kubectl get componentstatuses
# (deprecated in newer versions, use endpoint health checks)

# Check API server health
kubectl get --raw='/healthz'

# Check etcd health
kubectl get --raw='/healthz/etcd'
```

## Setting Up Cluster Monitoring

For ongoing cluster monitoring, deploy Prometheus + Grafana:

```yaml
# Add to your monitoring stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

After deployment, access Grafana dashboards via Portainer by navigating to the monitoring namespace and finding the Grafana service.

## Conclusion

Portainer's Kubernetes dashboard provides a quick overview of cluster health and resource utilization. For day-to-day operations, use the dashboard to spot nodes under stress, identify resource-heavy namespaces, and navigate to specific workloads. For deeper monitoring, complement Portainer with Prometheus and Grafana deployed in the cluster.
