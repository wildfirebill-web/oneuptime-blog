# How to Set Up GPU Containers for ML Workloads in Portainer - Workloads

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GPU, NVIDIA, Portainer, Machine Learning, Docker, CUDA, Deep Learning

Description: Configure Portainer to deploy containers with NVIDIA GPU access for machine learning workloads, including the NVIDIA Container Toolkit setup and resource allocation.

---

Running ML training and inference workloads in containers with GPU access requires the NVIDIA Container Toolkit and proper configuration in Docker and Portainer. This guide walks through the complete setup.

## Prerequisites

- Host with NVIDIA GPU (GTX 1080 or newer, Quadro, A-series, H-series)
- Ubuntu 20.04/22.04 or compatible Linux distribution
- NVIDIA drivers installed on the host

## Step 1: Install NVIDIA Container Toolkit on the Host

```bash
# Add NVIDIA Container Toolkit repository

curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit

# Configure Docker to use NVIDIA runtime
sudo nvidia-ctk runtime configure --runtime=docker

# Restart Docker
sudo systemctl restart docker

# Verify GPU is accessible to containers
docker run --rm --gpus all nvidia/cuda:12.3.0-base-ubuntu22.04 nvidia-smi
```

## Step 2: Enable GPU Resources in Portainer

Portainer reads the Docker daemon's resource capabilities. After installing the NVIDIA Container Toolkit, GPU resources will appear in Portainer's container creation form under **Runtime & Resources > GPUs**.

For stack deployments, use the `deploy.resources.reservations.devices` syntax:

```yaml
# ml-training-stack.yml
version: "3.8"

services:
  ml-trainer:
    image: nvcr.io/nvidia/pytorch:24.01-py3
    # Grant access to all GPUs
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - training-data:/data
      - model-output:/output
    restart: unless-stopped

volumes:
  training-data:
  model-output:
```

## Step 3: Run a Training Job

The following Compose definition runs a PyTorch training job as a one-off container via Portainer:

```yaml
# pytorch-training-job.yml
version: "3.8"

services:
  trainer:
    image: nvcr.io/nvidia/pytorch:24.01-py3
    command: python /workspace/train.py --epochs 50 --batch-size 128
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1    # Reserve exactly 1 GPU
              capabilities: [gpu]
    volumes:
      - /opt/ml/train.py:/workspace/train.py:ro
      - /opt/ml/data:/workspace/data:ro
      - /opt/ml/models:/workspace/models
    environment:
      - CUDA_VISIBLE_DEVICES=0
```

A sample training script using the GPU:

```python
# train.py - runs inside the container
import torch
import torch.nn as nn

# Automatically use GPU if available
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(f"Training on: {device}")
print(f"GPU: {torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'N/A'}")

# Simple model - move to GPU
model = nn.Sequential(
    nn.Linear(784, 256),
    nn.ReLU(),
    nn.Linear(256, 10)
).to(device)

# Training loop (abbreviated)
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
criterion = nn.CrossEntropyLoss()
```

## Step 4: Monitor GPU Usage

Monitor GPU utilization from within a running container:

```bash
# Run nvidia-smi inside the container from Portainer's console tab
nvidia-smi

# For continuous monitoring
nvidia-smi dmon -s u -d 1
```

Or deploy a GPU metrics exporter for Prometheus:

```yaml
  dcgm-exporter:
    image: nvcr.io/nvidia/k8s/dcgm-exporter:3.3.0-3.2.0-ubuntu22.04
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    ports:
      - "9400:9400"
```

## Step 5: Multi-GPU Workloads

For multi-GPU training (e.g., distributed PyTorch with DDP):

```yaml
  distributed-trainer:
    image: nvcr.io/nvidia/pytorch:24.01-py3
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 4       # Reserve 4 GPUs
              capabilities: [gpu]
    environment:
      - NCCL_DEBUG=INFO      # Enable NCCL debugging for multi-GPU
    ipc: host                # Required for shared memory between GPU processes
    ulimits:
      memlock: -1
      stack: 67108864
```

## Summary

Setting up GPU containers in Portainer requires the NVIDIA Container Toolkit on the host, and then using the `devices` reservation syntax in your compose stacks. Once configured, Portainer gives you a practical interface for deploying, monitoring, and managing GPU-accelerated ML workloads.
