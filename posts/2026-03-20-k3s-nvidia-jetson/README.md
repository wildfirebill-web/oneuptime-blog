# How to Install K3s on NVIDIA Jetson

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Kubernetes, NVIDIA, Jetson, GPU, Edge AI, ARM64

Description: Learn how to install K3s on NVIDIA Jetson devices and configure GPU access for AI/ML workloads at the edge.

## Introduction

NVIDIA Jetson devices are powerful ARM64 edge computing platforms with integrated GPU capabilities (CUDA, TensorRT). Combining K3s with NVIDIA Jetson enables AI/ML inference workloads to run at the edge as containerized Kubernetes workloads. This guide covers installation on Jetson Nano, Xavier NX, and AGX Orin.

## Supported Jetson Devices

| Device | CPU | GPU | RAM | Best For |
|--------|-----|-----|-----|---------|
| Jetson Nano | 4x Cortex-A57 | 128-core Maxwell | 4GB | Learning, light inference |
| Xavier NX | 6x Carmel ARM64 | 384-core Volta | 8/16GB | Production inference |
| AGX Orin | 12x Cortex-A78AE | 2048-core Ampere | 32/64GB | Heavy AI workloads |

## Prerequisites

- JetPack 5.x or 6.x installed on the Jetson
- Internet connectivity (or pre-downloaded images for air-gap)
- NVIDIA Container Runtime installed (included in JetPack 5+)

## Step 1: Verify JetPack and NVIDIA Runtime

```bash
# Check JetPack version

cat /etc/nv_tegra_release

# Verify NVIDIA container runtime is available
nvidia-ctk --version

# Verify CUDA is accessible
nvcc --version

# Test GPU access
sudo docker run --rm --runtime=nvidia --gpus all \
    nvcr.io/nvidia/l4t-base:r35.3.1 nvidia-smi
```

## Step 2: Enable cgroup Memory

```bash
# Check current boot parameters
cat /boot/extlinux/extlinux.conf

# Edit the boot config to add cgroup parameters
sudo nano /boot/extlinux/extlinux.conf

# Find the APPEND line and add:
# cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory

# Example APPEND line:
# APPEND ${cbootargs} root=/dev/mmcblk0p1 rw rootwait cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory

# Reboot
sudo reboot
```

After reboot, verify cgroups are enabled:

```bash
cat /proc/cgroups | grep memory
# Expect 1 in the last column (enabled)
```

## Step 3: Disable Swap

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify
free -h
```

## Step 4: Configure NVIDIA Container Runtime for containerd

K3s uses containerd. Configure the NVIDIA runtime for containerd:

```bash
# Configure the NVIDIA container toolkit for containerd
sudo nvidia-ctk runtime configure --runtime=containerd --set-as-default

# Verify the configuration
sudo cat /etc/containerd/config.toml | grep -A 10 "nvidia"
```

Or manually configure the K3s containerd template:

```bash
# Create a containerd config for K3s with NVIDIA support
sudo mkdir -p /var/lib/rancher/k3s/agent/etc/containerd/

sudo tee /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl > /dev/null <<'EOF'
[plugins.opt]
  path = "{{ .NodeConfig.Containerd.Opt }}"

[plugins.cri]
  stream_server_address = "127.0.0.1"
  stream_server_port = "10010"

{{- if .IsRunningInUserNS }}
  disable_cgroup = true
  disable_apparmor = true
  restrict_oom_score_adj = true
{{ end -}}

[plugins.cri.containerd]
  default_runtime_name = "nvidia"

  [plugins.cri.containerd.runtimes.nvidia]
    runtime_type = "io.containerd.runc.v2"
    [plugins.cri.containerd.runtimes.nvidia.options]
      BinaryName = "/usr/bin/nvidia-container-runtime"
      SystemdCgroup = true

  [plugins.cri.containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
    [plugins.cri.containerd.runtimes.runc.options]
      SystemdCgroup = true
EOF
```

## Step 5: Install K3s

```bash
sudo mkdir -p /etc/rancher/k3s

sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
# K3s server configuration for NVIDIA Jetson
token: "JetsonClusterToken"
tls-san:
  - $(hostname -I | awk '{print $1}')
  - $(hostname)

# Kubelet configuration for Jetson
kubelet-arg:
  - "max-pods=110"
  - "kube-reserved=cpu=500m,memory=1Gi"
  - "system-reserved=cpu=500m,memory=512Mi"
  - "eviction-hard=memory.available<200Mi,nodefs.available<5%"

# Node labels for GPU workload targeting
node-label:
  - "device-type=jetson"
  - "accelerator=nvidia-gpu"
  - "nvidia.com/gpu=true"
EOF

# Install K3s
curl -sfL https://get.k3s.io | sudo sh -

# Wait for startup
sudo systemctl status k3s
```

## Step 6: Install the NVIDIA Device Plugin

The NVIDIA device plugin allows Kubernetes to schedule GPU resources:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Create the NVIDIA device plugin DaemonSet
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.1/nvidia-device-plugin.yml

# Verify the plugin is running
kubectl -n kube-system get pods -l name=nvidia-device-plugin-ds

# Verify GPU resources are visible
kubectl get nodes -o json | jq '.items[].status.capacity'
# Should show: "nvidia.com/gpu": "1" (or more for multi-GPU)
```

## Step 7: Deploy an AI Inference Workload

```yaml
# object-detection.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: object-detection
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: object-detection
  template:
    metadata:
      labels:
        app: object-detection
    spec:
      # Target Jetson nodes with GPU
      nodeSelector:
        accelerator: nvidia-gpu
      containers:
        - name: inference
          image: nvcr.io/nvidia/l4t-ml:r35.3.1-py3
          command: ["sleep", "infinity"]
          # Request GPU resource
          resources:
            requests:
              cpu: "500m"
              memory: "2Gi"
              nvidia.com/gpu: 1
            limits:
              cpu: "2"
              memory: "4Gi"
              nvidia.com/gpu: 1
          volumeMounts:
            - name: dshm
              mountPath: /dev/shm
      volumes:
        - name: dshm
          emptyDir:
            medium: Memory
            sizeLimit: 1Gi
```

```bash
kubectl apply -f object-detection.yaml

# Verify the pod has GPU access
kubectl exec -it $(kubectl get pods -l app=object-detection -o jsonpath='{.items[0].metadata.name}') -- nvidia-smi
```

## Step 8: Configure GPU Time-Slicing (Optional)

For devices with multiple GPU workloads:

```yaml
# gpu-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: kube-system
data:
  any: |-
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        replicas: 4
```

## Monitoring GPU on Jetson

```bash
# Monitor GPU usage on the Jetson
sudo tegrastats

# Or from a pod
kubectl run gpu-monitor \
    --image=nvcr.io/nvidia/l4t-base:r35.3.1 \
    --restart=Never \
    --overrides='{"spec":{"nodeSelector":{"accelerator":"nvidia-gpu"},"containers":[{"name":"gpu-monitor","image":"nvcr.io/nvidia/l4t-base:r35.3.1","resources":{"limits":{"nvidia.com/gpu":"1"}},"command":["nvidia-smi","dmon"]}]}}'
```

## Conclusion

Installing K3s on NVIDIA Jetson devices creates a powerful edge AI platform that combines Kubernetes orchestration with GPU-accelerated inference. The key steps are configuring the NVIDIA container runtime for containerd, enabling cgroup memory, and deploying the NVIDIA device plugin to expose GPU resources to workloads. Once configured, you can deploy TensorRT, CUDA, or any GPU-accelerated application as standard Kubernetes workloads with GPU resource requests.
