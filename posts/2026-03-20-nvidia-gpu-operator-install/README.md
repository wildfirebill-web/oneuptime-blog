# How to Install NVIDIA GPU Operator in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, nvidia, gpu-operator, kubernetes, helm

Description: Detailed installation guide for the NVIDIA GPU Operator in Rancher-managed Kubernetes clusters.
## Introduction

The NVIDIA GPU Operator automates the management of all NVIDIA software components needed to provision GPUs in Kubernetes—drivers, container toolkit, device plugin, DCGM exporter, and more. This guide covers a complete installation in a Rancher environment.

## Architecture

The GPU Operator consists of several components:
- **NVIDIA Driver**: Kernel module for GPU hardware
- **Container Toolkit**: Runtime integration (nvidia-container-runtime)
- **Device Plugin**: Exposes GPUs as Kubernetes resources
- **DCGM Exporter**: GPU metrics for Prometheus
- **Node Feature Discovery**: Labels nodes with GPU capabilities
- **GPU Feature Discovery**: Detailed GPU feature labeling

## Prerequisites

```bash
# Verify GPU hardware is present on worker nodes
lspci | grep -i nvidia

# Check kernel version (must be 4.15+)
uname -r

# Verify containerd is the container runtime
kubectl get nodes -o wide | head -5
```

## Step 1: Configure Node Taints for GPU Nodes

```bash
# Taint GPU nodes to prevent non-GPU workloads
kubectl taint nodes gpu-node-01   nvidia.com/gpu=present:NoSchedule

# Add informational labels
kubectl label nodes gpu-node-01   accelerator=nvidia-tesla-a100   gpu-count=8
```

## Step 2: Add Helm Repository

```bash
# Add NVIDIA NGC Helm registry
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

# Verify repository is available
helm search repo nvidia/gpu-operator --versions | head -10
```

## Step 3: Create Namespace and Configuration

```bash
# Create namespace
kubectl create namespace gpu-operator

# Create imagePullSecret if using private registry
kubectl create secret docker-registry ngc-secret   --docker-server=nvcr.io   --docker-username='$oauthtoken'   --docker-password=YOUR_NGC_API_KEY   --namespace gpu-operator
```

## Step 4: Install with Custom Values

```yaml
# gpu-operator-values.yaml
operator:
  defaultRuntime: containerd   # or docker

driver:
  enabled: true
  version: "535.104.12"       # Pin driver version for stability
  repository: nvcr.io/nvidia
  image: driver
  rdma:
    enabled: false             # Enable for InfiniBand workloads
    useHostMofed: false

toolkit:
  enabled: true
  version: v1.14.3-centos7    # Match your OS

devicePlugin:
  enabled: true
  version: v0.14.1
  config:
    name: time-slicing-config  # For GPU sharing
    default: all

dcgmExporter:
  enabled: true
  version: 3.2.5-3.1.8-ubuntu20.04
  serviceMonitor:
    enabled: true              # Prometheus scraping

gfd:                           # GPU Feature Discovery
  enabled: true
  version: v0.8.1

migManager:
  enabled: false               # Enable for A100/H100 MIG

nodeStatusExporter:
  enabled: true

sandboxWorkloads:
  enabled: false

psp:
  enabled: false               # Disable for Kubernetes 1.25+
```

```bash
# Install with values file
helm install gpu-operator nvidia/gpu-operator   --namespace gpu-operator   --values gpu-operator-values.yaml   --version v23.9.0   --wait   --timeout 10m
```

## Step 5: Configure Time-Slicing (Optional)

Allow multiple workloads to share a single GPU:

```yaml
# time-slicing-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  all: |
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        renameByDefault: false
        failRequestsGreaterThanOne: false
        resources:
        - name: nvidia.com/gpu
          replicas: 4          # 4 virtual GPUs per physical GPU
```

```bash
kubectl apply -f time-slicing-config.yaml
```

## Step 6: Verify Installation

```bash
#!/bin/bash
# verify-gpu-operator.sh

echo "=== GPU Operator Verification ==="

# Check operator pod
kubectl get pods -n gpu-operator -l app=gpu-operator

# Check all daemonsets are running on GPU nodes
echo ""
echo "DaemonSet status:"
kubectl get daemonsets -n gpu-operator

# Check node labels applied by GFD
echo ""
echo "GPU node labels:"
kubectl get nodes -l nvidia.com/gpu.present=true   -o custom-columns='NAME:.metadata.name,GPU:.status.allocatable[nvidia\.com/gpu]'

# Run validation job
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/gpu-operator/main/tests/gpu-test.yaml
kubectl wait pod/cuda-vector-add --for=condition=Succeeded --timeout=60s
kubectl logs cuda-vector-add
```

## Step 7: Configure Prometheus Monitoring

```yaml
# gpu-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nvidia-dcgm-exporter
  namespace: gpu-operator
  labels:
    release: prometheus    # Match your Prometheus release label
spec:
  selector:
    matchLabels:
      app: nvidia-dcgm-exporter
  endpoints:
  - port: gpu-metrics
    interval: 30s
    path: /metrics
```

## Troubleshooting Common Issues

```bash
# Issue: Driver pod stuck in Init state
kubectl describe pod nvidia-driver-daemonset-xxx -n gpu-operator
# Check: kernel headers installed, secure boot disabled

# Install kernel headers
sudo apt-get install -y linux-headers-$(uname -r)

# Disable secure boot (in BIOS/UEFI)

# Issue: Device plugin not showing GPUs
kubectl logs nvidia-device-plugin-xxx -n gpu-operator
# Check: NVIDIA driver is loaded
nvidia-smi && lsmod | grep nvidia
```

## Conclusion

The NVIDIA GPU Operator simplifies GPU management in Kubernetes significantly. Once installed, it handles all the complex driver and runtime configuration automatically, allowing developers to request GPUs with simple resource specifications. Regular updates to the GPU Operator keep drivers and tools current with minimal manual intervention.
