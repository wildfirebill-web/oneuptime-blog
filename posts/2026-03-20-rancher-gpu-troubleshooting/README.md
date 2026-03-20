# How to Troubleshoot GPU Issues in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, GPU, Troubleshooting, NVIDIA, Kubernetes, Debugging

Description: Comprehensive troubleshooting guide for resolving common GPU-related issues in Rancher Kubernetes clusters.

## Introduction

GPU issues in Rancher can stem from hardware, drivers, container runtime, or Kubernetes configuration. This guide provides systematic troubleshooting steps for the most common GPU problems.

## Common Symptoms and Causes

| Symptom | Likely Cause |
|---------|-------------|
| Pods stuck in Pending | No GPU resources available |
| CUDA errors in logs | Driver version mismatch |
| nvidia-smi fails | Driver not loaded |
| GPU not visible in pod | Container toolkit misconfigured |
| Poor performance | Memory oversubscription |

## Step 1: Verify GPU Hardware Detection

```bash
# On the GPU node

# Check PCI device
lspci | grep -i nvidia

# Check kernel module
lsmod | grep nvidia

# If module not loaded, load it
sudo modprobe nvidia
sudo modprobe nvidia_uvm

# Check dmesg for GPU errors
sudo dmesg | grep -i nvidia | tail -20

# Run nvidia-smi
nvidia-smi
```

## Step 2: Check GPU Operator Pod Status

```bash
# List all GPU operator pods
kubectl get pods -n gpu-operator -o wide

# Check for pods not Running
kubectl get pods -n gpu-operator | grep -v Running

# Describe problematic pod
kubectl describe pod nvidia-driver-daemonset-xxx -n gpu-operator

# Check driver pod logs
kubectl logs -n gpu-operator   nvidia-driver-daemonset-xxx   --previous   # Previous container logs if crashed
```

## Step 3: Debug CUDA Container Issues

```bash
# Test CUDA access in a debug container
kubectl run cuda-debug   --image=nvcr.io/nvidia/cuda:12.2.0-base-ubuntu22.04   --rm -it   --restart=Never   --limits=nvidia.com/gpu=1   -- nvidia-smi

# If it fails, check the device plugin
kubectl logs -n gpu-operator   $(kubectl get pods -n gpu-operator -l app=nvidia-device-plugin-daemonset -o name | head -1)

# Test container toolkit
kubectl run toolkit-test   --image=nvcr.io/nvidia/cuda:12.2.0-base-ubuntu22.04   --rm -it   --restart=Never   --limits=nvidia.com/gpu=1   -- sh -c "ls -la /dev/nvidia*"
```

## Step 4: Diagnose Pod Scheduling Failures

```bash
# Check why pod is in Pending state
kubectl describe pod gpu-workload | grep -A20 Events

# Common messages:
# "0/3 nodes are available: 3 Insufficient nvidia.com/gpu"
#   -> No GPU nodes have capacity

# "0/3 nodes are available: 3 node(s) had taint"
#   -> Missing toleration for GPU taint

# Check GPU resource availability
kubectl get nodes   -o custom-columns='NAME:.metadata.name,GPU:.status.allocatable[nvidia\.com/gpu]'

# Check what GPU resources are currently used
kubectl get pods -A -o json | jq '
  .items[] | 
  select(.spec.containers[].resources.limits["nvidia.com/gpu"] != null) |
  {pod: .metadata.name, namespace: .metadata.namespace, 
   gpu: .spec.containers[].resources.limits["nvidia.com/gpu"]}'
```

## Step 5: Fix Driver Version Mismatches

```bash
# Check installed driver version
nvidia-smi --query-gpu=driver_version --format=csv,noheader

# Check CUDA version required by your image
kubectl exec gpu-workload -- nvcc --version

# Check compatibility matrix at:
# https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/

# Update GPU Operator to use correct driver version
helm upgrade gpu-operator nvidia/gpu-operator   --namespace gpu-operator   --reuse-values   --set driver.version="535.104.12"   # Pin to specific version
```

## Step 6: Reset Stuck GPU Resources

```bash
# If GPU resources appear stuck/unavailable after pod deletion
# Restart device plugin daemonset
kubectl rollout restart daemonset nvidia-device-plugin-daemonset   -n gpu-operator

# If that doesn't help, restart the node (last resort)
kubectl drain gpu-node-01 --ignore-daemonsets --delete-emptydir-data
# Restart node
ssh admin@gpu-node-01 sudo reboot
# After node comes back
kubectl uncordon gpu-node-01
```

## Step 7: Debug DCGM Exporter Issues

```bash
# Check DCGM exporter is running
kubectl get pods -n gpu-operator -l app=nvidia-dcgm-exporter

# View DCGM metrics directly
kubectl port-forward -n gpu-operator   $(kubectl get pods -n gpu-operator -l app=nvidia-dcgm-exporter -o name | head -1)   9400:9400 &

curl -s http://localhost:9400/metrics | grep -v "^#" | head -20

# Check for DCGM errors
kubectl logs -n gpu-operator   $(kubectl get pods -n gpu-operator -l app=nvidia-dcgm-exporter -o name | head -1)   | grep -i error
```

## Step 8: Collect Diagnostic Information

```bash
#!/bin/bash
# gpu-diagnostics.sh - Collect comprehensive diagnostics

echo "=== GPU Diagnostics Report ===" > /tmp/gpu-diag.txt
echo "Date: $(date)" >> /tmp/gpu-diag.txt

echo "" >> /tmp/gpu-diag.txt
echo "=== Node Info ===" >> /tmp/gpu-diag.txt
kubectl get nodes -o wide >> /tmp/gpu-diag.txt

echo "" >> /tmp/gpu-diag.txt
echo "=== GPU Operator Pods ===" >> /tmp/gpu-diag.txt
kubectl get pods -n gpu-operator -o wide >> /tmp/gpu-diag.txt

echo "" >> /tmp/gpu-diag.txt
echo "=== GPU Resources ===" >> /tmp/gpu-diag.txt
kubectl get nodes   -o custom-columns='NAME:.metadata.name,GPU:.status.allocatable[nvidia\.com/gpu]'   >> /tmp/gpu-diag.txt

echo "Diagnostics saved to /tmp/gpu-diag.txt"
```

## Conclusion

GPU troubleshooting in Rancher follows a systematic approach: verify hardware, check the GPU Operator components, test container access, and diagnose scheduling issues. Most problems stem from driver mismatches, container runtime configuration, or resource exhaustion. Use the diagnostic commands in this guide to quickly identify and resolve issues.
