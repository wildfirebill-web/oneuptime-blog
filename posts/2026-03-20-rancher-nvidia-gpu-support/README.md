# How to Configure NVIDIA GPU Support in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, NVIDIA, GPU, Kubernetes, Machine-learning

Description: Complete guide to enabling NVIDIA GPU support in Rancher clusters for machine learning and GPU-accelerated workloads.

## Introduction

NVIDIA GPUs dramatically accelerate machine learning, scientific computing, and graphics workloads. Rancher makes it straightforward to configure GPU support across your Kubernetes clusters using the NVIDIA GPU Operator.

## Prerequisites

- Rancher v2.6+
- Kubernetes cluster with NVIDIA GPU nodes
- Nodes running Ubuntu 20.04/22.04 or RHEL 8/9
- NVIDIA GPUs: Tesla, A100, H100, RTX series

## Step 1: Verify GPU Hardware

```bash
# Check GPU is detected by the OS

lspci | grep -i nvidia

# Expected output example:
# 00:1e.0 3D controller: NVIDIA Corporation A100 80GB PCIe [10de:20b5]

# Check NVIDIA driver (if pre-installed)
nvidia-smi
```

## Step 2: Label GPU Nodes

```bash
# Label nodes that have GPUs
kubectl label nodes gpu-node-01   nvidia.com/gpu.present=true   node-role.kubernetes.io/gpu=true

# Verify labels
kubectl get nodes --show-labels | grep nvidia
```

## Step 3: Install NVIDIA GPU Operator via Helm

```bash
# Add NVIDIA Helm repository
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

# Install GPU Operator
helm install gpu-operator nvidia/gpu-operator   --namespace gpu-operator   --create-namespace   --set driver.enabled=true   --set toolkit.enabled=true   --set devicePlugin.enabled=true   --set dcgmExporter.enabled=true   --set gfd.enabled=true   --version v23.9.0
```

## Step 4: Configure Node Feature Discovery

```yaml
# node-feature-discovery.yaml
apiVersion: nfd.k8s-sigs.io/v1alpha1
kind: NodeFeatureDiscovery
metadata:
  name: nfd-instance
  namespace: gpu-operator
spec:
  operand:
    image: registry.k8s.io/nfd/node-feature-discovery:v0.14.0
    imagePullPolicy: IfNotPresent
  workerConfig:
    configData: |
      core:
        sleepInterval: 60s
      sources:
        pci:
          deviceClassWhitelist:
            - "0300"
            - "0302"
          deviceLabelFields:
            - vendor
```

## Step 5: Verify GPU Operator Installation

```bash
# Check all GPU operator pods are running
kubectl get pods -n gpu-operator

# Expected pods:
# gpu-operator-xxx                Running
# nvidia-driver-daemonset-xxx     Running  (on each GPU node)
# nvidia-container-toolkit-xxx    Running  (on each GPU node)
# nvidia-device-plugin-xxx        Running  (on each GPU node)
# nvidia-dcgm-exporter-xxx        Running  (on each GPU node)

# Check GPU is available as a resource
kubectl get nodes -o json | jq '.items[] | {
  name: .metadata.name,
  gpus: .status.allocatable["nvidia.com/gpu"]
}'
```

## Step 6: Test GPU Access

```yaml
# gpu-test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  restartPolicy: Never
  containers:
  - name: cuda-container
    image: nvidia/cuda:12.2.0-base-ubuntu22.04
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1    # Request 1 GPU
      requests:
        nvidia.com/gpu: 1
```

```bash
kubectl apply -f gpu-test-pod.yaml
kubectl logs gpu-test
# Should show nvidia-smi output with GPU details
```

## Step 7: Configure RuntimeClass

```yaml
# nvidia-runtime-class.yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
scheduling:
  nodeSelector:
    nvidia.com/gpu.present: "true"
```

## Step 8: Deploy GPU-Accelerated Workload

```yaml
# ml-training-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ml-training
spec:
  template:
    spec:
      runtimeClassName: nvidia    # Use NVIDIA runtime
      restartPolicy: Never
      containers:
      - name: trainer
        image: pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime
        command: ["python", "train.py"]
        resources:
          limits:
            nvidia.com/gpu: 2    # Request 2 GPUs
            memory: "32Gi"
            cpu: "8"
          requests:
            nvidia.com/gpu: 2
            memory: "16Gi"
            cpu: "4"
        env:
        - name: CUDA_VISIBLE_DEVICES
          value: "all"
      nodeSelector:
        nvidia.com/gpu.present: "true"
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
```

## Monitoring GPU Usage

```bash
# View GPU metrics via DCGM exporter
kubectl port-forward svc/nvidia-dcgm-exporter   -n gpu-operator 9400:9400 &

curl -s http://localhost:9400/metrics | grep -E "DCGM_FI_DEV_GPU_UTIL|DCGM_FI_DEV_MEM_COPY_UTIL"
```

## Conclusion

NVIDIA GPU support in Rancher enables powerful GPU-accelerated workloads with minimal configuration. The GPU Operator handles driver installation, device plugin configuration, and monitoring setup automatically. Once configured, requesting GPUs in Kubernetes workloads is as simple as adding resource limits to your pod specifications.
