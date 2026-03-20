# How to Configure GPU Time-Slicing in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, gpu, time-slicing, kubernetes, nvidia, sharing

Description: Guide to configuring NVIDIA GPU time-slicing in Rancher to allow multiple workloads to share a single GPU.
## Introduction

GPU time-slicing enables multiple pods to share a single physical GPU by allocating compute time in slices. Unlike MIG (which provides hard partitioning), time-slicing provides soft partitioning with shared memory—suitable for development and lighter workloads.

## Time-Slicing vs MIG Comparison

| Feature | Time-Slicing | MIG |
|---------|-------------|-----|
| Memory isolation | No (shared) | Yes (dedicated) |
| Compute isolation | No (shared) | Yes (dedicated) |
| GPU generations | All NVIDIA | A100, H100 only |
| Oversubscription | Possible | Not possible |
| Use case | Dev/testing | Production |

## Step 1: Create Time-Slicing ConfigMap

```yaml
# time-slicing-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  any: |
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        renameByDefault: false
        failRequestsGreaterThanOne: false
        resources:
        - name: nvidia.com/gpu
          replicas: 4           # Each GPU appears as 4 virtual GPUs
  
  # Node-specific configurations
  a100-40gb: |
    version: v1
    sharing:
      timeSlicing:
        resources:
        - name: nvidia.com/gpu
          replicas: 8           # 8 virtual GPUs per A100-40GB
  
  rtx-3090: |
    version: v1
    sharing:
      timeSlicing:
        resources:
        - name: nvidia.com/gpu
          replicas: 2           # 2 virtual GPUs per RTX 3090
```

```bash
kubectl apply -f time-slicing-config.yaml
```

## Step 2: Configure GPU Operator to Use Time-Slicing

```yaml
# gpu-operator-update.yaml - Patch existing installation
devicePlugin:
  config:
    name: time-slicing-config     # Reference the ConfigMap
    default: any                  # Default config to apply
```

```bash
# Update GPU Operator with time-slicing config
helm upgrade gpu-operator nvidia/gpu-operator   --namespace gpu-operator   --reuse-values   --set devicePlugin.config.name=time-slicing-config   --set devicePlugin.config.default=any
```

## Step 3: Apply Time-Slicing to Nodes

```bash
# Label a node to use time-slicing config
kubectl label node gpu-node-01   nvidia.com/device-plugin.config=a100-40gb   --overwrite

# Label all GPU nodes with default config
kubectl get nodes -l nvidia.com/gpu.present=true   -o name | xargs -I{} kubectl label {}   nvidia.com/device-plugin.config=any   --overwrite

# Verify time-slicing is applied
kubectl get nodes -l nvidia.com/gpu.present=true   -o custom-columns='NAME:.metadata.name,GPU:.status.allocatable[nvidia\.com/gpu]'
# With 1 physical GPU and replicas=4, should show 4 virtual GPUs
```

## Step 4: Deploy Multiple Workloads on Same GPU

```yaml
# inference-service-1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-service-1
  namespace: dev-team-a
spec:
  replicas: 1
  selector:
    matchLabels:
      app: inference-1
  template:
    metadata:
      labels:
        app: inference-1
    spec:
      nodeSelector:
        nvidia.com/gpu.present: "true"
      containers:
      - name: inference
        image: nvcr.io/nvidia/tritonserver:23.09-py3
        resources:
          limits:
            nvidia.com/gpu: "1"   # Gets 1 time-sliced virtual GPU
---
# inference-service-2.yaml - Shares the same physical GPU
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-service-2
  namespace: dev-team-b
spec:
  replicas: 1
  selector:
    matchLabels:
      app: inference-2
  template:
    metadata:
      labels:
        app: inference-2
    spec:
      nodeSelector:
        nvidia.com/gpu.present: "true"
      containers:
      - name: inference
        image: nvcr.io/nvidia/tritonserver:23.09-py3
        resources:
          limits:
            nvidia.com/gpu: "1"   # Shares physical GPU via time-slicing
```

## Step 5: Validate Time-Slicing

```bash
# After deploying multiple pods, verify they all land on the same physical GPU
kubectl get pods -A -o wide | grep -E "inference-1|inference-2"

# Both should be on same node
# Check GPU processes running on the node
ssh admin@gpu-node-01 "sudo nvidia-smi"
# Should show multiple processes using the same GPU

# Check virtual GPU count
kubectl describe node gpu-node-01 | grep nvidia.com/gpu
# Capacity:    nvidia.com/gpu: 4    (4 virtual GPUs)
# Allocatable: nvidia.com/gpu: 4
```

## Monitoring Time-Sliced GPU Usage

```bash
# Since all virtual GPUs map to the same physical GPU,
# DCGM metrics show aggregate utilization
kubectl exec -n gpu-operator   $(kubectl get pods -n gpu-operator -l app=nvidia-dcgm-exporter -o name | head -1)   -- nvidia-smi dmon -s u -d 2

# Check which pods are using GPU
sudo nvidia-smi pmon -d 2
```

## Conclusion

GPU time-slicing in Rancher enables GPU resource sharing without requiring specialized hardware like A100 or H100. It is particularly valuable for development environments where strict isolation is not required, but multiple developers need GPU access. Configure replicas based on your workload characteristics and available GPU memory.
