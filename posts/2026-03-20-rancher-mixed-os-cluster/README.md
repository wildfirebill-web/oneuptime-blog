# How to Configure Mixed Linux and Windows Clusters in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Windows, Linux, Mixed OS, Hybrid Cluster

Description: Configure and operate Rancher Kubernetes clusters with both Linux and Windows worker nodes, managing workload placement, networking, and node lifecycle in a hybrid environment.

## Introduction

Mixed OS clusters allow organizations to consolidate Linux and Windows workloads under a single Kubernetes management plane. Linux nodes handle Linux-native microservices while Windows nodes run .NET Framework, IIS, and Windows-specific applications. This guide covers the architecture, configuration, and operational practices for hybrid clusters in Rancher.

## Prerequisites

- RKE2 cluster with Linux control plane
- Windows Server 2019/2022 worker nodes
- kubectl and Rancher UI access
- Flannel CNI (recommended for Windows compatibility)

## Step 1: Cluster Architecture

```
Mixed OS Cluster Architecture:
├── Control Plane (Linux only)
│   ├── etcd: 3x Linux nodes
│   └── API Server, Controller Manager, Scheduler: Linux nodes
│
├── Linux Worker Nodes
│   ├── Microservices (Go, Node.js, Python, Java)
│   ├── Databases (PostgreSQL, Redis, MongoDB)
│   └── Message queues (Kafka, RabbitMQ)
│
└── Windows Worker Nodes
    ├── .NET Framework applications (IIS)
    ├── .NET 8 applications
    └── Windows Services

Note: Windows nodes CANNOT run:
- Control plane components (API server, etcd, scheduler)
- Most CNI plugins (only Flannel/Calico Windows edition)
- Most system daemonsets
```

## Step 2: Add Windows Nodes to Existing Linux Cluster

```bash
# Check existing cluster is Linux-only
kubectl get nodes -o wide

# Verify CNI is Windows-compatible (Flannel)
kubectl get configmap rke2-cfg -n kube-system -o yaml | grep cni

# On new Windows node (as Administrator):
# Download and run RKE2 Windows agent
# See: rancher-windows-workers post for detailed steps
```

## Step 3: Configure Workload Placement

```yaml
# Ensure Linux workloads stay on Linux nodes
# Best practice: add explicit nodeSelector to all workloads

# Linux workload - explicit Linux selector
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linux-microservice
  namespace: production
spec:
  template:
    spec:
      # Explicit Linux node selector
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: service
          image: registry.example.com/linux-service:v1.0
---
# Windows workload
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows-app
  namespace: production
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      tolerations:
        - key: os
          value: windows
          effect: NoSchedule
      containers:
        - name: app
          image: registry.example.com/windows-app:v1.0
```

## Step 4: Configure System DaemonSets for Mixed OS

```yaml
# System DaemonSets (like node-exporter, Fluent Bit) should target Linux only
# Add nodeSelector to system DaemonSets

kubectl patch daemonset node-exporter \
  -n cattle-monitoring-system \
  --type=json \
  -p='[{
    "op": "add",
    "path": "/spec/template/spec/nodeSelector",
    "value": {"kubernetes.io/os": "linux"}
  }]'

kubectl patch daemonset prometheus-node-exporter \
  -n cattle-monitoring-system \
  --type=json \
  -p='[{
    "op": "add",
    "path": "/spec/template/spec/nodeSelector",
    "value": {"kubernetes.io/os": "linux"}
  }]'
```

## Step 5: Cross-OS Service Communication

```yaml
# Linux pod calling Windows service
# Windows service deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows-api
  namespace: production
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      containers:
        - name: api
          image: registry.example.com/windows-api:v1.0
          ports:
            - containerPort: 8080
---
# Service (OS-agnostic - any pod can call this)
apiVersion: v1
kind: Service
metadata:
  name: windows-api
  namespace: production
spec:
  selector:
    app: windows-api  # Selects Windows pods
  ports:
    - port: 80
      targetPort: 8080
```

```python
# Linux Python service calling Windows API
import requests

# This works seamlessly - Kubernetes handles routing across OS
response = requests.get("http://windows-api.production.svc.cluster.local/api/data")
```

## Step 6: Configure Node Pools and Labels

```bash
# Add descriptive labels to differentiate node types
kubectl label node win-node-01 \
  node-type=windows-worker \
  workload-profile=dotnet \
  windows-version=2022

kubectl label node linux-worker-01 \
  node-type=linux-worker \
  workload-profile=general

# Use node pools via Rancher UI for organized management
# Rancher allows separate node pools per OS with different instance types
```

## Step 7: Mixed OS Monitoring Setup

```yaml
# Deploy Windows Exporter only on Windows nodes
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: windows-exporter
  namespace: cattle-monitoring-system
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      tolerations:
        - key: os
          value: windows
          effect: NoSchedule
---
# Linux node exporter DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: cattle-monitoring-system
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      # No toleration needed - Linux is default
```

## Step 8: Upgrade Strategy for Mixed OS Clusters

```bash
# Windows nodes must be upgraded separately from Linux nodes
# RKE2 supports mixed-version clusters temporarily during upgrade

# Step 1: Upgrade control plane (Linux)
# Through Rancher UI: Cluster > Edit > Kubernetes Version

# Step 2: Upgrade Linux worker nodes
# Rancher drains and upgrades each node sequentially

# Step 3: Upgrade Windows worker nodes
# Windows nodes require manual upgrade - drain first
kubectl drain win-node-01 --ignore-daemonsets --delete-emptydir-data

# On Windows node, upgrade RKE2
# Uninstall old version
& "C:\rke2\bin\rke2.exe" agent service --delete
# Install new version
Invoke-WebRequest -Uri "https://github.com/rancher/rke2/releases/download/v1.29.0+rke2r1/rke2-windows-amd64.zip" -OutFile rke2.zip
# Follow installation steps
# ...

kubectl uncordon win-node-01
```

## Conclusion

Mixed OS Kubernetes clusters in Rancher provide a unified management platform for heterogeneous workloads. The fundamental principle is always specifying `kubernetes.io/os` node selectors in all pod specifications—this ensures workloads never accidentally schedule on the wrong OS. Cross-OS service communication works transparently through Kubernetes services, enabling Linux and Windows pods to call each other via standard DNS names. Monitor both Linux and Windows nodes through their respective exporters, and plan separate upgrade windows for Linux and Windows nodes since they require different procedures.
