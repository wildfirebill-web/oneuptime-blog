# How to Configure Mixed Linux and Windows Clusters in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Mixed Cluster, Window, Linux, Kubernetes, Node Selectors

Description: Configure and manage mixed Linux and Windows Kubernetes clusters in Rancher with proper workload scheduling, resource organization, and OS-aware configurations.

## Introduction

Mixed Linux/Windows clusters in Rancher allow organizations to run both modern cloud-native Linux workloads and Windows-specific legacy applications in the same Kubernetes cluster. Proper configuration ensures workloads land on the correct OS without manual intervention.

## Cluster Requirements

- Linux nodes: Control plane, etcd, and Linux worker nodes
- Windows nodes: Worker nodes only (Windows cannot run control plane components)
- Network plugin: Flannel VXLAN (both OS supported)

## Step 1: Label Nodes by OS

Kubernetes automatically labels nodes with `kubernetes.io/os`:

```bash
# Verify OS labels

kubectl get nodes -L kubernetes.io/os,kubernetes.io/arch

# Add custom labels for Windows version targeting
kubectl label node win-worker-1 \
  windows-version=2022 \
  workload-class=windows
kubectl label node win-worker-2 \
  windows-version=2022 \
  workload-class=windows
```

## Step 2: Taint Windows Nodes

Prevent Linux workloads from accidentally scheduling on Windows nodes:

```bash
# Taint Windows nodes
kubectl taint nodes win-worker-1 win-worker-2 \
  os=windows:NoSchedule
```

## Step 3: Configure Linux Workloads to Avoid Windows Nodes

Linux deployments should explicitly select Linux nodes:

```yaml
# All Linux deployments should include this
spec:
  nodeSelector:
    kubernetes.io/os: linux    # Ensure Linux workloads don't land on Windows
```

## Step 4: Configure Windows Workloads

```yaml
# Windows workload configuration
spec:
  nodeSelector:
    kubernetes.io/os: windows
  tolerations:
    - key: os
      operator: Equal
      value: windows
      effect: NoSchedule
```

## Step 5: Configure Shared Services

Some services (like monitoring agents and DNS helpers) need to run on both OS types:

```yaml
# DaemonSet that runs on all nodes
spec:
  template:
    spec:
      # No OS-specific nodeSelector - runs on all nodes
      tolerations:
        - key: os    # Tolerate the Windows taint
          operator: Exists
          effect: NoSchedule
```

## Step 6: Resource Quotas by OS

```yaml
# Separate resource quotas for Linux and Windows workloads
# Create separate namespaces for OS-specific workloads
apiVersion: v1
kind: Namespace
metadata:
  name: windows-workloads
  labels:
    workload-os: windows
---
apiVersion: v1
kind: Namespace
metadata:
  name: linux-workloads
  labels:
    workload-os: linux
```

## Step 7: Networking Considerations

```yaml
# Services work identically across OS types
# A Linux pod can call a Windows service and vice versa
apiVersion: v1
kind: Service
metadata:
  name: windows-api
  namespace: windows-workloads
spec:
  selector:
    app: windows-api
  ports:
    - port: 80
```

## Conclusion

Mixed Linux/Windows clusters in Rancher work reliably with proper node taints and workload selectors. The OS abstraction layer in Kubernetes means Linux and Windows workloads can communicate seamlessly through Services. The main operational challenge is ensuring new deployments include the correct OS node selector to prevent unexpected scheduling.
